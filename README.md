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
