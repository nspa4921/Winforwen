import { LightningElement, track } from 'lwc';
import subscribeToNewsletter from '@salesforce/apex/NCNewsletterSignupService.subscribeToNewsletter';
import unsubscribeFromNewsletter from '@salesforce/apex/NCNewsletterSignupService.unsubscribeFromNewsletter';

export default class NewsletterSignup extends LightningElement {
    @track email = '';
    @track browserId = '';
    @track message = '';
    @track error = '';
    @track loading = false;
    @track showModal = false;

    handleEmailChange(event) {
        this.email = event.target.value;
    }
    handleBrowserIdChange(event) {
        this.browserId = event.target.value;
        console.log(this.browserId, "br id");
    }
    handleSubmit() {
        this.loading = true;
        this.message = '';
        this.error = '';
        subscribeToNewsletter({ email: this.email, browserId: this.browserId })
            .then(result => {
                this.message = result;
                this.loading = false;
                this.showModal = true;
                // Email is NOT sent immediately. Scheduled Apex will handle it.
            })
            .catch(error => {
                this.error = error.body ? error.body.message : error.message;
                this.loading = false;
            });
    }

    closeModal() {
        this.showModal = false;
    }

    handleUnsubscribe(event) {
        event.preventDefault();
        this.loading = true;
        this.message = '';
        this.error = '';
        unsubscribeFromNewsletter({ email: this.email })
            .then(result => {
                this.message = result;
                this.loading = false;
            })
            .catch(error => {
                this.error = error.body ? error.body.message : error.message;
                this.loading = false;
            });
    }
}

public without sharing class NCNewsletterSignupService {
  @AuraEnabled
  public static String subscribeToNewsletter(String email, String browserId) {
    if (String.isBlank(email)) {
      throw new AuraHandledException('Email is required.');
    }

    // Try to find Individual by browserId (if provided)
    Individual individual = null;
    if (!String.isBlank(browserId)) {
      List<Individual> individuals = [
        SELECT Id
        FROM Individual
        WHERE Browser_Id__c = :browserId
        LIMIT 1
      ];
      if (!individuals.isEmpty()) {
        individual = individuals[0];
      }
    }

    List<Lead> existingLeads = [
      SELECT Id, Email
      FROM Lead
      WHERE Email = :email
      LIMIT 1
    ];

    if (!existingLeads.isEmpty()) {
      // If lead already exists, do not create
      return 'You are already subscribed.';
    }

    // Create new Lead, link to Individual if found
    Lead newLead = new Lead(
      FirstName = 'Newsletter',
      LastName = 'Subscriber',
      Email = email,
      Company = 'Newsletter',
      Status = 'Subscribed',
      LeadSource = 'Newsletter',
      IndividualId = (individual != null ? individual.Id : null) // Lookup(Individual)
      // Newsletter_Email_Sent__c left as default (false)
    );
    insert newLead;

    // If Individual exists, update related Assessments/Responses
    if (individual != null) {
      List<Assessment> assessments = [
        SELECT Id
        FROM Assessment
        WHERE Individual__c = :individual.Id
      ];
      List<AssessmentQuestionResponse> toUpdate = new List<AssessmentQuestionResponse>();
      for (Assessment ass : assessments) {
        for (AssessmentQuestionResponse resp : [
          SELECT Id
          FROM AssessmentQuestionResponse
          WHERE AssessmentId = :ass.Id
        ]) {
          resp.Lead__c = newLead.Id; // Lookup(Lead)
          toUpdate.add(resp);
        }
      }
      if (!toUpdate.isEmpty()) {
        update toUpdate;
      }
    }
    return 'You are now subscribed to our newsletter';
  }

  // Sends a single email to the given address with subject and body
  @AuraEnabled
  public static Boolean sendSingleEmail(String toEmail, String subject, String body) {
    if (String.isBlank(toEmail)) {
      throw new AuraHandledException('Email address is required.');
    }
    Messaging.reserveSingleEmailCapacity(1);
    EmailTemplate[] templates = [SELECT Id FROM EmailTemplate WHERE Name = 'NovoCare Newsletter Welcome Email' LIMIT 1];
    Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
    mail.setToAddresses(new List<String>{ toEmail });
    if (!templates.isEmpty()) {
      List<Lead> leads = [SELECT Id FROM Lead WHERE Email = :toEmail LIMIT 1];
      if (!leads.isEmpty()) {
        mail.setTargetObjectId(leads[0].Id);
        mail.setTemplateId(templates[0].Id);
        mail.setSaveAsActivity(false);
      } else {
        mail.setSubject(subject);
        mail.setPlainTextBody(body);
      }
    } else {
      mail.setSubject(subject);
      mail.setPlainTextBody(body);
    }
    OrgWideEmailAddress[] orgEmails = [SELECT Id FROM OrgWideEmailAddress WHERE Address = 'nemo.spaske@gmail.com' LIMIT 1];
    if (!orgEmails.isEmpty()) {
        mail.setOrgWideEmailAddressId(orgEmails[0].Id);
    }
    Messaging.SendEmailResult[] results = Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ mail });
    return results != null && results.size() > 0 && results[0].isSuccess();
  }

  @AuraEnabled
  public static String unsubscribeFromNewsletter(String email) {
    if (String.isBlank(email)) {
      throw new AuraHandledException('Email is required.');
    }
    List<Lead> leads = [SELECT Id, Status, UnsubscribedDate__c FROM Lead WHERE Email = :email LIMIT 1];
    if (leads.isEmpty()) {
      return 'No subscription found for this email.';
    }
    Lead lead = leads[0];
    lead.Status = 'Unsubscribed';
    lead.UnsubscribedDate__c = System.now();
    update lead;
    EmailTemplate[] templates = [SELECT Id FROM EmailTemplate WHERE Name = 'NovoCare Newsletter Unsubscribe Email' LIMIT 1];
    if (!templates.isEmpty()) {
      Messaging.reserveSingleEmailCapacity(1);
      Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
      mail.setTargetObjectId(lead.Id);
      mail.setTemplateId(templates[0].Id);
      mail.setSaveAsActivity(false);
      OrgWideEmailAddress[] orgEmails = [SELECT Id FROM OrgWideEmailAddress WHERE Address = 'nemo.spaske@gmail.com' LIMIT 1];
      if (!orgEmails.isEmpty()) {
        mail.setOrgWideEmailAddressId(orgEmails[0].Id);
      }
      Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ mail });
    }
    return 'You have been unsubscribed.';
  }
}



global class NCNewsletterScheduledEmail implements Schedulable {
    global void execute(SchedulableContext sc) {
        // Find leads created more than 1 hour ago, but not yet sent newsletter email
        Datetime oneHourAgo = System.now().addHours(-1);
        List<Lead> leads = [
            SELECT Id, Email, IndividualId, Newsletter_Email_Sent__c
            FROM Lead
            WHERE CreatedDate <= :oneHourAgo
            AND Newsletter_Email_Sent__c = false
            AND Status = 'Subscribed'
            AND Email != null
        ];

        for (Lead lead : leads) {
            Boolean hasAssessment = false;
            if (lead.IndividualId != null) {
                List<Assessment> assessments = [
                    SELECT Id FROM Assessment WHERE Individual__c = :lead.IndividualId LIMIT 1
                ];
                hasAssessment = !assessments.isEmpty();
            }

            if (hasAssessment) {
                NCNewsletterSignupService.sendSingleEmail(
                    lead.Email,
                    'Assessment Created - Welcome!',
                    'Thank you for subscribing and completing your assessment!'
                );
            } else {
                NCNewsletterSignupService.sendSingleEmail(
                    lead.Email,
                    'Welcome to NovoCare Newsletter!',
                    'Thank you for subscribing! Complete your assessment for more personalized content.'
                );
            }
            lead.Newsletter_Email_Sent__c = true;
        }
        if (!leads.isEmpty()) {
            update leads;
        }
    }
}

///////////////////////////////////  --------------------------- ///////////////////////////////////////


global with sharing class NCNewsletterScheduledEmail implements Schedulable {

    global void execute(SchedulableContext sc) {

        // Window: [now-60m , now-45m)
        Datetime windowStart = System.now().addMinutes(-60);
        Datetime windowEnd   = System.now().addMinutes(-45);

        // Pick leads to email
        List<Lead> leads = [
            SELECT Id, Email, Status, Individual__c, Newsletter_Email_Sent__c
            FROM Lead
            WHERE CreatedDate >= :windowStart
              AND CreatedDate  < :windowEnd
              AND Email != null
              AND Newsletter_Email_Sent__c = false
              AND Status != 'Unsubscribed'
            LIMIT 5000
        ];
        if (leads.isEmpty()) return;

        // Build lookup sets/maps
        Set<Id> individualIds = new Set<Id>();
        for (Lead l : leads) if (l.Individual__c != null) individualIds.add(l.Individual__c);

        // Do they have assessments?
        // Use whichever object you truly track completion on:
        //  A) Assessment__c by Individual__c
        Map<Id, Boolean> individualHasAssessment = new Map<Id, Boolean>();
        if (!individualIds.isEmpty()) {
            for (AggregateResult ar : [
                SELECT Individual__c ind, COUNT(Id) c
                FROM Assessment__c
                WHERE Individual__c IN :individualIds
                GROUP BY Individual__c
            ]) {
                individualHasAssessment.put((Id)ar.get('ind'), ((Decimal)ar.get('c')) > 0);
            }
        }

        // Prepare emails (use templates by Name)
        Id orgWideId = NCNewsletterSignupService.getOrgWideIdByAddress('nemo.spasak@gmail.com'); // <= change to yours

        // Cache template Ids once
        Id templateA = NCNewsletterSignupService.getTemplateIdByName('NovoCare Newsletter – Assessment Complete'); // Email A
        Id templateB = NCNewsletterSignupService.getTemplateIdByName('NovoCare Newsletter – Welcome / Complete Assessment'); // Email B

        List<Messaging.SingleEmailMessage> msgs = new List<Messaging.SingleEmailMessage>();
        for (Lead l : leads) {
            Boolean hasAsm = (l.Individual__c != null) && individualHasAssessment.get(l.Individual__c) == true;
            Id tpl = hasAsm ? templateA : templateB;
            Messaging.SingleEmailMessage m = NCNewsletterSignupService.buildTemplatedEmail(l.Id, l.Email, tpl, orgWideId);
            if (m != null) msgs.add(m);
        }

        if (!msgs.isEmpty()) {
            List<Messaging.SendEmailResult> results = Messaging.sendEmail(msgs, false);

            // Mark success + collect failures
            List<Lead> toUpdate = new List<Lead>();
            for (Integer i = 0; i < results.size(); i++) {
                if (results[i].isSuccess()) {
                    Lead l = leads[i].clone(false, true, false, false);
                    l.Id = leads[i].Id;
                    l.put('Newsletter_Email_Sent__c', true);
                    if (Schema.sObjectType.Lead.fields.getMap().containsKey('Newsletter_Email_Sent_Date__c')) {
                        l.put('Newsletter_Email_Sent_Date__c', System.now());
                    }
                    toUpdate.add(l);
                } else {
                    // Log but don't throw – avoid killing the job
                    System.debug(LoggingLevel.WARN,
                        'Email failed for Lead ' + leads[i].Id + ': ' + results[i].getErrors()[0].getMessage());
                }
            }
            if (!toUpdate.isEmpty()) update toUpdate;
        }
    }

    // Helper to schedule: “run every 15 minutes”
    public static void scheduleEvery15Min() {
        // CRON: second minute hour dayOfMonth month dayOfWeek year(optional)
        // 0 0/15 * * * ?  -> every 15 minutes
        System.schedule('Newsletter job scheduler (15m)', '0 0/15 * * * ?', new NCNewsletterScheduledEmail());
    }
}


#‐---------

public with sharing class NCNewsletterSignupService {

    // existing methods...

    public static Id getOrgWideIdByAddress(String address) {
        OrgWideEmailAddress owea = [
            SELECT Id FROM OrgWideEmailAddress WHERE Address = :address LIMIT 1
        ];
        return owea.Id;
    }

    public static Id getTemplateIdByName(String templateName) {
        EmailTemplate t = [
            SELECT Id FROM EmailTemplate WHERE Name = :templateName LIMIT 1
        ];
        return t.Id;
    }

    // Build a SingleEmailMessage using a classic email template + Lead/Contact as target
    public static Messaging.SingleEmailMessage buildTemplatedEmail(Id whoId, String toEmail, Id templateId, Id orgWideId) {
        if (whoId == null || toEmail == null || templateId == null) return null;

        Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
        mail.setTargetObjectId(whoId);     // must be a Lead or Contact when using a template
        mail.setTemplateId(templateId);
        mail.setOrgWideEmailAddressId(orgWideId);
        mail.setSaveAsActivity(false);     // optional
        mail.setToAddresses(new String[] { toEmail });
        return mail;
    }
}

#-☆☆☆☆



# Salesforce DX Project: Next Steps

Now that you’ve created a Salesforce DX project, what’s next? Here are some documentation resources to get you started.

## How Do You Plan to Deploy Your Changes?

Do you want to deploy a set of changes, or create a self-contained application? Choose a [development model](https://developer.salesforce.com/tools/vscode/en/user-guide/development-models).

## Configure Your Salesforce DX Project

The `sfdx-project.json` file contains useful configuration information for your project. See [Salesforce DX Project Configuration](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ws_config.htm) in the _Salesforce DX Developer Guide_ for details about this file.

## Read All About It

- [Salesforce Extensions Documentation](https://developer.salesforce.com/tools/vscode/)
- [Salesforce CLI Setup Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_setup.meta/sfdx_setup/sfdx_setup_intro.htm)
- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)
- [Salesforce CLI Command Reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference.htm)
