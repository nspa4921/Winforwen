Super, ovo je zaokružena verzija koja radi tačno šta želiš:
– jedan Schedulable na 15 min cilja Lead-ove kreirane između 60 i 45 min ranije, koji imaju Status = 'Subscribed', nisu poslali mejl (Newsletter_Email_Sent__c = false) i imaju email.
– Ako postoji bar 1 Assessment__c za njihovog Individual → šalje Email A; u suprotnom Email B.
– Koristi OrgWideEmailAddress (kao u tvojoj metodi) i email template-ove.
– Sve je bulk-ifikovano i sa ispravnim tipovima/generics.


---

LWC (sitna ispravka – uklonjen slučajni backtick)

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

  handleEmailChange(event) { this.email = event.target.value; }
  handleBrowserIdChange(event) { this.browserId = event.target.value; }

  handleSubmit() {
    this.loading = true; this.message = ''; this.error = '';
    subscribeToNewsletter({ email: this.email, browserId: this.browserId })
      .then(result => { this.message = result; this.showModal = true; })
      .catch(error => { this.error = error.body ? error.body.message : error.message; })
      .finally(() => { this.loading = false; });
  }

  closeModal() { this.showModal = false; }

  handleUnsubscribe(event) {
    event.preventDefault();
    this.loading = true; this.message = ''; this.error = '';
    unsubscribeFromNewsletter({ email: this.email })
      .then(result => { this.message = result; })
      .catch(error => { this.error = error.body ? error.body.message : error.message; })
      .finally(() => { this.loading = false; });
  }
}


---

NCNewsletterSignupService.cls (servis + helperi)

> Napomena o poljima:
• Lead.IndividualId – standard lookup na Individual (ostavljam ga jer ga već koristiš).
• Lead.Status – koristi vrednosti 'Subscribed' i 'Unsubscribed' kao kod tebe.
• Lead.Newsletter_Email_Sent__c (Checkbox) i Lead.UnsubscribedDate__c (Datetime) – tvoja custom polja.
• Assessment__c.Individual__c – pretpostavka (u tvojoj bazi je objekat Assessment/Assessment__c; ispod koristi Assessment__c – promeni naziv ako ti je drugačiji).



public with sharing class NCNewsletterSignupService {

    @AuraEnabled
    public static String subscribeToNewsletter(String email, String browserId) {
        if (String.isBlank(email)) {
            throw new AuraHandledException('Email is required.');
        }

        // Poveži Individual po browserId (ako postoji)
        Individual individual = null;
        if (!String.isBlank(browserId)) {
            List<Individual> inds = [
                SELECT Id
                FROM Individual
                WHERE Browser_Id__c = :browserId
                LIMIT 1
            ];
            if (!inds.isEmpty()) individual = inds[0];
        }

        // Ne dupliraj Lead po emailu
        List<Lead> existingLeads = [
            SELECT Id FROM Lead WHERE Email = :email LIMIT 1
        ];
        if (!existingLeads.isEmpty()) {
            return 'You are already subscribed.';
        }

        // Kreiraj Lead
        Lead newLead = new Lead(
            FirstName = 'Newsletter',
            LastName  = 'Subscriber',
            Email     = email,
            Company   = 'Newsletter',
            Status    = 'Subscribed',
            LeadSource= 'Newsletter',
            IndividualId = (individual != null ? individual.Id : null) // standard field
            // Newsletter_Email_Sent__c ostaje default false
        );
        insert newLead;

        // (Opcionalno) Ako postoji Individual, poveži postojeće odgovore na novi Lead
        if (individual != null) {
            List<Assessment__c> assessments = [
                SELECT Id FROM Assessment__c WHERE Individual__c = :individual.Id
            ];

            List<AssessmentQuestionResponse__c> toUpdate = new List<AssessmentQuestionResponse__c>();
            for (Assessment__c a : assessments) {
                for (AssessmentQuestionResponse__c r : [
                    SELECT Id FROM AssessmentQuestionResponse__c WHERE Assessment__c = :a.Id
                ]) {
                    r.Lead__c = newLead.Id;
                    toUpdate.add(r);
                }
            }
            if (!toUpdate.isEmpty()) update toUpdate;
        }
        return 'You are now subscribed to our newsletter';
    }

    // ---------- Email helperi (Org-Wide + Template) ----------

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

    // Sastavi SingleEmailMessage za Lead sa template-om
    public static Messaging.SingleEmailMessage buildTemplatedEmail(Id leadId, String toEmail, Id templateId, Id orgWideId) {
        if (leadId == null || String.isBlank(toEmail) || templateId == null) return null;
        Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
        mail.setTargetObjectId(leadId);           // Lead/Contact obavezno za template
        mail.setTemplateId(templateId);
        mail.setToAddresses(new String[]{ toEmail });
        if (orgWideId != null) mail.setOrgWideEmailAddressId(orgWideId);
        mail.setSaveAsActivity(false);
        return mail;
    }

    // (Zadržiš i postojeći unsubscribe)

    @AuraEnabled
    public static String unsubscribeFromNewsletter(String email) {
        if (String.isBlank(email)) {
            throw new AuraHandledException('Email is required.');
        }
        List<Lead> leads = [SELECT Id, Status, UnsubscribedDate__c FROM Lead WHERE Email = :email LIMIT 1];
        if (leads.isEmpty()) return 'No subscription found for this email.';

        Lead l = leads[0];
        l.Status = 'Unsubscribed';
        l.UnsubscribedDate__c = System.now();
        update l;

        // Pošalji potvrdu o odjavi (ako imaš template)
        if (Schema.getGlobalDescribe().get('EmailTemplate') != null) {
            List<EmailTemplate> templates = [SELECT Id FROM EmailTemplate WHERE Name = 'NovoCare Newsletter Unsubscribe Email' LIMIT 1];
            if (!templates.isEmpty()) {
                Messaging.reserveSingleEmailCapacity(1);
                Id owea = getOrgWideIdByAddress('nemo.spaske@gmail.com'); // promeni na svoju adresu
                Messaging.SingleEmailMessage mail = buildTemplatedEmail(l.Id, email, templates[0].Id, owea);
                Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ mail });
            }
        }
        return 'You have been unsubscribed.';
    }
}


---

NCNewsletterScheduledEmail.cls (svakih 15 min, prozor 60–45 min)

> Zameni IMENA TEMPLATE-OVA ispod prema stvarnim nazivima u tvojoj org-u:
TPL_ASSESSMENT_DONE = „Email A”,
TPL_WELCOME_COMPLETE_ASM = „Email B”.



global with sharing class NCNewsletterScheduledEmail implements Schedulable {

    global void execute(SchedulableContext sc) {

        // Obradi lead-ove kreirane pre tačno ~1h (prozor 60–45 min)
        Datetime windowStart = System.now().addMinutes(-60);
        Datetime windowEnd   = System.now().addMinutes(-45);

        List<Lead> leads = [
            SELECT Id, Email, Status, IndividualId, Newsletter_Email_Sent__c
            FROM Lead
            WHERE CreatedDate >= :windowStart
              AND CreatedDate  < :windowEnd
              AND Newsletter_Email_Sent__c = false
              AND Status = 'Subscribed'
              AND Email != null
            LIMIT 5000
        ];
        if (leads.isEmpty()) return;

        // Sakupi IndividualId za lookup
        Set<Id> individualIds = new Set<Id>();
        for (Lead l : leads) if (l.IndividualId != null) individualIds.add(l.IndividualId);

        // Mapiraj: Individual -> ima li Assessment
        Map<Id, Boolean> hasAsmByIndividual = new Map<Id, Boolean>();
        if (!individualIds.isEmpty()) {
            for (AggregateResult ar : [
                SELECT Individual__c ind, COUNT(Id) c
                FROM Assessment__c
                WHERE Individual__c IN :individualIds
                GROUP BY Individual__c
            ]) {
                hasAsmByIndividual.put((Id)ar.get('ind'), ((Decimal)ar.get('c')) > 0);
            }
        }

        // Priprema template-ova i OrgWide
        final String TPL_ASSESSMENT_DONE       = 'NovoCare Newsletter – Assessment Complete';          // Email A
        final String TPL_WELCOME_COMPLETE_ASM  = 'NovoCare Newsletter – Welcome / Complete Assessment'; // Email B

        Id tplA = NCNewsletterSignupService.getTemplateIdByName(TPL_ASSESSMENT_DONE);
        Id tplB = NCNewsletterSignupService.getTemplateIdByName(TPL_WELCOME_COMPLETE_ASM);
        Id owea = NCNewsletterSignupService.getOrgWideIdByAddress('nemo.spaske@gmail.com'); // promeni na svoju OWEA adresu

        List<Messaging.SingleEmailMessage> msgs = new List<Messaging.SingleEmailMessage>();
        for (Lead l : leads) {
            Boolean hasAsm = l.IndividualId != null && hasAsmByIndividual.get(l.IndividualId) == true;
            Id tpl = hasAsm ? tplA : tplB;
            Messaging.SingleEmailMessage m = NCNewsletterSignupService.buildTemplatedEmail(l.Id, l.Email, tpl, owea);
            if (m != null) msgs.add(m);
        }

        if (!msgs.isEmpty()) {
            List<Messaging.SendEmailResult> res = Messaging.sendEmail(msgs, false);

            // Obeleži poslate
            List<Lead> toUpd = new List<Lead>();
            for (Integer i = 0; i < res.size(); i++) {
                if (res[i].isSuccess()) {
                    Lead l = new Lead(Id = leads[i].Id);
                    l.Newsletter_Email_Sent__c = true;
                    toUpd.add(l);
                } else {
                    // ne bacaj exception da ne ubiješ job; samo log
                    System.debug(LoggingLevel.WARN,
                        'Email failed for Lead ' + leads[i].Id + ': ' +
                        (res[i].getErrors().isEmpty() ? 'Unknown' : res[i].getErrors()[0].getMessage()));
                }
            }
            if (!toUpd.isEmpty()) update toUpd;
        }
    }

    // Jedan raspored na 15 minuta
    public static void scheduleEvery15Min() {
        System.schedule('Newsletter job scheduler (15m)', '0 0/15 * * * ?', new NCNewsletterScheduledEmail());
    }
}


---

Kako da zakažeš (jedan jedini job)

Setup → Apex Classes → Schedule Apex
Name: Newsletter job scheduler (15m)
CRON: 0 0/15 * * * ?


---

Tip

Greška iz mejla EMAIL_ADDRESS_BOUNCED znači da je adresa obeležena kao bounced na Lead/Contact-u. Na zapisu klikni Reset Bounced Email (ili očisti polja EmailBouncedDate/EmailBouncedReason) pre testiranja.

Ako ti je objekat/field API ime drugačije (npr. Assessment umesto Assessment__c, AssessmentId/Assessment__c u odgovorima), samo reci tačna imena i prepakujem gore kod tačno na tvoja polja.



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
