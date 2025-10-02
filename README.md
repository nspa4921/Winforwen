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

@IsTest
private class NCNewsletterSignupService_Test {
    // Simple email mock: mark all messages as "sent successfully"
    private class AllSuccessEmailMock implements Messaging.SendEmailMock {
        public Messaging.SendEmailResult[] respond(Messaging.SendEmailRequest[] reqs) {
            Messaging.SendEmailResult[] results = new Messaging.SendEmailResult[reqs.size()];
            for (Integer i = 0; i < reqs.size(); i++) {
                Messaging.SendEmailResult r = new Messaging.SendEmailResult();
                r.isSuccess = true;
                r.success = true;
                results[i] = r;
            }
            return results;
        }
    }

    // Optional: a mock that fails the 2nd email — uncomment if you want to cover the "failure" branch too
    /*
    private class OneFailureEmailMock implements Messaging.SendEmailMock {
        public Messaging.SendEmailResult[] respond(Messaging.SendEmailRequest[] reqs) {
            Messaging.SendEmailResult[] results = new Messaging.SendEmailResult[reqs.size()];
            for (Integer i = 0; i < reqs.size(); i++) {
                Messaging.SendEmailResult r = new Messaging.SendEmailResult();
                if (i == 1) {
                    r.isSuccess = false;
                    r.success = false;
                    r.errors = new Messaging.SendEmailError[] {
                        new Messaging.SendEmailError('Simulated failure')
                    };
                } else {
                    r.isSuccess = true;
                    r.success = true;
                }
                results[i] = r;
            }
            return results;
        }
    }
    */

    @IsTest
    static void test_SendEmails_UpdatesLeads() {
        // --- Test data ---
        List<Lead> leads = new List<Lead>{
            new Lead(LastName='L1', Company='C1', Email='l1@example.com', Newsletter_Email_Sent__c=false),
            new Lead(LastName='L2', Company='C2', Email='l2@example.com', Newsletter_Email_Sent__c=false),
            // Ineligible lead (no email): should not be updated/sent
            new Lead(LastName='NoEmail', Company='C3', Newsletter_Email_Sent__c=false)
        };
        insert leads;

        // --- Mock email send so nothing real goes out ---
        Test.setMock(Messaging.SendEmailMock.class, new AllSuccessEmailMock());
        // If you want to also hit the "failure" branch, swap the mock:
        // Test.setMock(Messaging.SendEmailMock.class, new OneFailureEmailMock());

        Test.startTest();
        // ❗️Replace this with YOUR real entry method that runs the logic you pasted.
        // Examples you might have:
        // NCNewsletterSignupService.run(); 
        // NCNewsletterSignupBatch.execute(...);
        // System.schedule('x','0 0 12 * * ?', new NCNewsletterSignupScheduler());
        NCNewsletterSignupService.sendPendingNewsletterEmails();
        Test.stopTest();

        // --- Verify results ---
        Map<Id, Lead> afterLeads = new Map<Id, Lead>([
            SELECT Id, Email, Newsletter_Email_Sent__c
            FROM Lead
            WHERE Id IN :leads
        ]);

        // Sent ones should be true
        System.assertEquals(true,  afterLeads.get(leads[0].Id).Newsletter_Email_Sent__c, 'Lead 1 should be marked sent');
        System.assertEquals(true,  afterLeads.get(leads[1].Id).Newsletter_Email_Sent__c, 'Lead 2 should be marked sent');

        // Ineligible one should remain false
        System.assertEquals(false, afterLeads.get(leads[2].Id).Newsletter_Email_Sent__c, 'Lead without email should not be marked sent');
    }
}

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
