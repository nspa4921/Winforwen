global with sharing class NCNewsletterScheduledEmail implements Schedulable {
  global void execute(SchedulableContext sc) {
    // Process leads created exactly ~1h ago (window 60–45 min)
    Datetime windowStart = System.now().addMinutes(-60);
    Datetime windowEnd = System.now().addMinutes(-45);

    System.debug(
      '🔍 Looking for leads created between ' +
        windowStart +
        ' and ' +
        windowEnd
    );

    List<Lead> leads = [
      SELECT
        Id,
        Email,
        Status,
        IndividualId,
        Newsletter_Email_Sent__c,
        Newsletter_Signup_Date__c
      FROM Lead
      WHERE
        Newsletter_Signup_Date__c >= :windowStart.date()
        AND Newsletter_Signup_Date__c < :windowEnd.date()
        AND Newsletter_Email_Sent__c = FALSE
        AND Status = 'Subscribed'
        AND Email != NULL
      LIMIT 5000
    ];

    System.debug('📋 Found ' + leads.size() + ' leads to process');
    if (leads.isEmpty()) {
      System.debug('ℹ️ No emails to send');
      return;
    }

    // Collect IndividualId for lookup
    Set<Id> individualIds = new Set<Id>();
    for (Lead l : leads) {
      if (l.IndividualId != null) {
        individualIds.add(l.IndividualId);
        System.debug(
          '📝 Processing Lead: ' +
            l.Id +
            ' (' +
            l.Email +
            ') created at: ' +
            l.Newsletter_Signup_Date__c
        );
      }
    }

    // Map: Individual -> has Assessment
    Map<Id, Boolean> hasAsmByIndividual = new Map<Id, Boolean>();
    if (!individualIds.isEmpty()) {
      for (AggregateResult ar : [
        SELECT Individual__c ind, COUNT(Id) c
        FROM Assessment
        WHERE Individual__c IN :individualIds
        GROUP BY Individual__c
      ]) {
        hasAsmByIndividual.put((Id) ar.get('ind'), ((Decimal) ar.get('c')) > 0);
      }
    }

    // Prepairing templates + OrgWide
    final String TPL_ASSESSMENT_DONE = 'NovoCare Newsletter Welcome Email'; // Email A JP25OB00165
    final String TPL_WELCOME_COMPLETE_ASM = 'NovoCare Newsletter Assessment Not Completed'; // Email B JP25OB00167

    Id tplA = NCNewsletterSignupService.getTemplateIdByName(
      TPL_ASSESSMENT_DONE
    );
    Id tplB = NCNewsletterSignupService.getTemplateIdByName(
      TPL_WELCOME_COMPLETE_ASM
    );
    Id owea = NCNewsletterSignupService.getOrgWideIdByAddress(
      'nemo.spaske@gmail.com'
    ); // cnahge before PROD!

    List<Messaging.SingleEmailMessage> msgs = new List<Messaging.SingleEmailMessage>();
    for (Lead l : leads) {
      Boolean hasAsm =
        l.IndividualId != null &&
        hasAsmByIndividual.get(l.IndividualId) == true;
      Id tpl = hasAsm ? tplA : tplB;
      Messaging.SingleEmailMessage m = NCNewsletterSignupService.buildTemplatedEmail(
        l.Id,
        l.Email,
        tpl,
        owea
      );
      if (m != null)
        msgs.add(m);
    }

    if (!msgs.isEmpty()) {
      System.debug('Attempting to send ' + msgs.size() + ' emails...');
      List<Messaging.SendEmailResult> res = Messaging.sendEmail(msgs, false);

      // Mark sent emails and log results
      List<Lead> toUpd = new List<Lead>();
      Integer successCount = 0;
      Integer failureCount = 0;

      for (Integer i = 0; i < res.size(); i++) {
        if (res[i].isSuccess()) {
          Lead l = new Lead(Id = leads[i].Id);
          l.Newsletter_Email_Sent__c = true;
          toUpd.add(l);
          successCount++;
          System.debug('✅ Email sent successfully to: ' + leads[i].Email);
        } else {
          failureCount++;
          System.debug(
            LoggingLevel.ERROR,
            '❌ Email failed for Lead ' +
              leads[i].Id +
              ' (' +
              leads[i].Email +
              '): ' +
              (res[i].getErrors().isEmpty()
                ? 'Unknown'
                : res[i].getErrors()[0].getMessage())
          );
        }
      }

      System.debug(
        '📧 Email Summary: ' +
          successCount +
          ' sent, ' +
          failureCount +
          ' failed'
      );

      if (!toUpd.isEmpty()) {
        update toUpd;
        System.debug('✅ Updated ' + toUpd.size() + ' leads as email sent');
      }
    } else {
      System.debug('ℹ️ No emails to send');
    }
  }
}


---//---

@isTest
private class NCNewsletterScheduledEmailTest {

    @isTest
    static void testExecute() {
        // Get Folder Id
        List<EmailTemplate> existingTemplates = [SELECT Id, FolderId FROM EmailTemplate LIMIT 1];
        Id emailFolderId;

        if (!existingTemplates.isEmpty()) {
            emailFolderId = existingTemplates[0].FolderId;
        } else {
            System.assert(false, 'No Email Templates found. Please create at least one Email Template in your org.');
            return;
        }

        // Create Email Templates
        EmailTemplate templateA = new EmailTemplate(
            Name = 'NovoCare Newsletter Welcome Email',
            DeveloperName = 'Newsletter_test_Template_A',
            TemplateType = 'text',
            Body = 'Test Email Template A',
            FolderId = emailFolderId
        );
        insert templateA;

        EmailTemplate templateB = new EmailTemplate(
            Name = 'NovoCare Newsletter Assessment Not Completed',
            DeveloperName = 'NovoCare_Newsletter_Unsubscribe_Email_Teamplte_B',
            TemplateType = 'text',
            Body = 'Test Email Template B',
            FolderId = emailFolderId
        );
        insert templateB;

        // Create an Individual
        Individual individual = new Individual(
            LastName = 'browser-1756721647940-qyh9hj8p8'
        );
        insert individual;

        DateTime createdAt = System.now().addMinutes(-50); // 50 minutes ago
        Date signupDate = createdAt.date();

        // Create a Lead with the necessary fields
        Lead leadWithAsm = new Lead(
            FirstName = 'Test',
            LastName = 'Lead',
            Company = 'Test Company',
            Email = 'test1@example.com',
            Status = 'Subscribed',
            Newsletter_Email_Sent__c = false,
            Newsletter_Signup_Date__c = signupDate,
            IndividualId = individual.Id
        );

        Lead leadWithoutAsm = new Lead(
            FirstName = 'Test',
            LastName = 'Lead',
            Company = 'Test Company',
            Email = 'test2@example.com',
            Status = 'Subscribed',
            Newsletter_Email_Sent__c = false,
            Newsletter_Signup_Date__c = signupDate,
            IndividualId = individual.Id
        );

        insert new List<Lead> { leadWithAsm, leadWithoutAsm };

        // Create an Assessment record for the leadWithAsm
        Assessment assessment = new Assessment(
            Individual__c = individual.Id,
            Name = 'Test Assessment',
            AssessmentStatus = 'Completed'
        );

        insert assessment;

        // Start the test
        Test.startTest();

        // Schedule the job
        NCNewsletterScheduledEmail scheduler = new NCNewsletterScheduledEmail();
        String sch = '0 0 0 * * ?'; // Run immediately
        String jobId = System.schedule('TestNewsletterScheduler', sch, scheduler);

        Test.stopTest();

        // Query the leads after the scheduled job has run
        List<Lead> updatedLeads = [SELECT Id, Newsletter_Email_Sent__c FROM Lead WHERE Id IN :new List<Id>{leadWithAsm.Id, leadWithoutAsm.Id}];

        // Assert outcomes
        // leadWithAsm should have Newsletter_Email_Sent__c = true
        // leadWithoutAsm should also have Newsletter_Email_Sent__c = true
        // Assuming both emails are valid and sent successfully
        System.assertEquals(true, updatedLeads[0].Newsletter_Email_Sent__c, 'Lead with assessment should have email sent.');
        System.assertEquals(true, updatedLeads[1].Newsletter_Email_Sent__c, 'Lead without assessment should have email sent.');
    }

    @isTest
    static void testNoLeadsToProcess() {
        // Start the test
        Test.startTest();

        // Schedule the job
        NCNewsletterScheduledEmail scheduler = new NCNewsletterScheduledEmail();
        String sch = '0 0 0 * * ?'; // Run immediately
        String jobId = System.schedule('TestNewsletterSchedulerNoLeads', sch, scheduler);

        Test.stopTest();

        // Since there are no leads, the logs should indicate no emails to send
        // You can check the debug logs to verify that it correctly logged no emails to send
    }
}

error:
System.DmlException: Insert failed. First exception on row 0; first error: MIXED_DML_OPERATION, DML operation on setup object is not permitted after you have updated a non-setup object (or vice versa): Lead, original object: EmailTemplate: []
Class.NCNewsletterScheduledEmailTest.testExecute: line 68, column 1
--//--

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
