<template>
  <section class="slds-grid slds-grid_vertical slds-align_absolute-center slds-p-around_x-large"
           style="min-height:60vh;">
    <div class="slds-box slds-theme_default slds-size_1-of-1 slds-medium-size_6-of-12 slds-large-size_4-of-12">
      <template if:true={loading}>
        <div class="slds-text-align_center slds-p-around_medium">
          <lightning-spinner alternative-text="Processing"></lightning-spinner>
          <p class="slds-m-top_medium">Processing your request…</p>
        </div>
      </template>

      <template if:false={loading}>
        <template if:true={success}>
          <div class="slds-text-align_center">
            <lightning-icon icon-name="utility:success" size="large"></lightning-icon>
            <h2 class="slds-text-heading_medium slds-m-top_medium">Unsubscribed</h2>
            <p class="slds-m-top_small">{message}</p>
          </div>
        </template>

        <template if:false={success}>
          <div class="slds-text-align_center">
            <lightning-icon icon-name="utility:error" size="large"></lightning-icon>
            <h2 class="slds-text-heading_medium slds-m-top_medium">Unable to unsubscribe</h2>
            <p class="slds-m-top_small">{message}</p>
          </div>
        </template>
      </template>
    </div>
  </section>
</template>


import { LightningElement, track } from 'lwc';
import unsubscribeUser from '@salesforce/apex/NewsletterUnsubscribeController.unsubscribeUser';

export default class Unsubscribe extends LightningElement {
  @track loading = true;
  @track success = false;
  @track message = '';

  connectedCallback() {
    // Čitanje tokena iz query string-a: /s/unsubscribe?token=XXXXX
    const params = new URLSearchParams(window.location.search);
    const token = params.get('token');

    if (!token) {
      this.loading = false;
      this.success = false;
      this.message = 'Missing token.';
      return;
    }

    unsubscribeUser({ token })
      .then((res) => {
        this.success = !!res?.success;
        this.message = res?.message || (this.success ? 'Unsubscribed.' : 'Invalid link.');
      })
      .catch(() => {
        this.success = false;
        this.message = 'Unexpected error.';
      })
      .finally(() => {
        this.loading = false;
      });
  }
}

Super — evo gotovog Apex + LWC rešenja za public “Unsubscribe” stranu na Experience Cloud-u.
Kada user kli
kne link iz emaila (.../s/unsubscribe?token=...), komponenta pročita token, pozove Apex i setuje Lead.Status = "Unsubscribed" (po želji i Email_Opt_Out = true). Prikaže se poruka o uspehu ili grešci.


---

Apex controller

classes/NewsletterUnsubscribeController.cls

public with sharing class NewsletterUnsubscribeController {

    public class UnsubscribeResult {
        @AuraEnabled public Boolean success;
        @AuraEnabled public String message;
    }

    @AuraEnabled(cacheable=false)
    public static UnsubscribeResult unsubscribeUser(String token) {
        UnsubscribeResult res = new UnsubscribeResult();

        if (String.isBlank(token)) {
            res.success = false;
            res.message = 'Missing token.';
            return res;
        }

        // Nađi lead po jedinstvenom tokenu
        List<Lead> leads = [
            SELECT Id, Status, Email_Opt_Out
            FROM Lead
            WHERE Newsletter_Token__c = :token
            LIMIT 1
        ];

        if (leads.isEmpty()) {
            res.success = false;
            res.message = 'Invalid or expired link.';
            return res;
        }

        Lead l = leads[0];

        // ❗ Uveri se da picklist sadrži vrednost "Unsubscribed"
        l.Status = 'Unsubscribed';
        // (opciono) takođe isključi email
        l.Email_Opt_Out = true;

        try {
            update l;
            res.success = true;
            res.message = 'You have been unsubscribed successfully.';
        } catch (Exception e) {
            res.success = false;
            res.message = 'Unexpected error while unsubscribing.';
        }
        return res;
    }
}

classes/NewsletterUnsubscribeController.cls-meta.xml

<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>60.0</apiVersion>
    <status>Active</status>
</ApexClass>


---

LWC: unsubscribe

lwc/unsubscribe/unsubscribe.html

<template>
  <section class="slds-grid slds-grid_vertical slds-align_absolute-center slds-p-around_x-large"
           style="min-height:60vh;">
    <div class="slds-box slds-theme_default slds-size_1-of-1 slds-medium-size_6-of-12 slds-large-size_4-of-12">
      <template if:true={loading}>
        <div class="slds-text-align_center slds-p-around_medium">
          <lightning-spinner alternative-text="Processing"></lightning-spinner>
          <p class="slds-m-top_medium">Processing your request…</p>
        </div>
      </template>

      <template if:false={loading}>
        <template if:true={success}>
          <div class="slds-text-align_center">
            <lightning-icon icon-name="utility:success" size="large"></lightning-icon>
            <h2 class="slds-text-heading_medium slds-m-top_medium">Unsubscribed</h2>
            <p class="slds-m-top_small">{message}</p>
          </div>
        </template>

        <template if:false={success}>
          <div class="slds-text-align_center">
            <lightning-icon icon-name="utility:error" size="large"></lightning-icon>
            <h2 class="slds-text-heading_medium slds-m-top_medium">Unable to unsubscribe</h2>
            <p class="slds-m-top_small">{message}</p>
          </div>
        </template>
      </template>
    </div>
  </section>
</template>

lwc/unsubscribe/unsubscribe.js

import { LightningElement, track } from 'lwc';
import unsubscribeUser from '@salesforce/apex/NewsletterUnsubscribeController.unsubscribeUser';

export default class Unsubscribe extends LightningElement {
  @track loading = true;
  @track success = false;
  @track message = '';

  connectedCallback() {
    // Čitanje tokena iz query string-a: /s/unsubscribe?token=XXXXX
    const params = new URLSearchParams(window.location.search);
    const token = params.get('token');

    if (!token) {
      this.loading = false;
      this.success = false;
      this.message = 'Missing token.';
      return;
    }

    unsubscribeUser({ token })
      .then((res) => {
        this.success = !!res?.success;
        this.message = res?.message || (this.success ? 'Unsubscribed.' : 'Invalid link.');
      })
      .catch(() => {
        this.success = false;
        this.message = 'Unexpected error.';
      })
      .finally(() => {
        this.loading = false;
      });
  }
}

lwc/unsubscribe/unsubscribe.js-meta.xml

<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
  <apiVersion>60.0</apiVersion>
  <isExposed>true</isExposed>
  <targets>
    <!-- Omogućava dodavanje u Experience Cloud Builder -->
    <target>lightningCommunity__Page</target>
    <target>lightningCommunity__Default</target>
    <!-- (opciono) za interne app stranice -->
    <target>lightning__AppPage</target>
  </targets>
</LightningComponentBundle>


---

Kako povezati sve u Experience Cloud-u

1. Deploy gore navedene fajlove.


2. Proveri da Lead Status picklist sadrži "Unsubscribed" (Setup → Object Manager → Lead → Fields → Lead Status).


3. Guest User Permissions (važno!):

Setup → Digital Experiences > All Sites → Workspace za tvoj sajt → Administration > Pages > Guest User Profile.

Daj Apex Class Access na NewsletterUnsubscribeController.

Daj Object Permissions na Lead: Read + Edit, i Field Level Security na Status, Newsletter_Token__c, Email_Opt_Out.



4. U Experience Builder-u:

Napravi novu stranu npr. “Unsubscribe” (URL: /unsubscribe).

Dodaj LWC unsubscribe na stranu, Publish.



5. U email template stavi link:

https://<your-community-domain>/s/unsubscribe?token={{Lead.Newsletter_Token__c}}

(ili {{Recipient.Newsletter_Token__c}} ako koristiš Recipient kontekst)




---

Brzi test

Uzmi Lead koji ima popunjen Newsletter_Token__c.

Otvori u browseru:
https://<your-community-domain>/s/unsubscribe?token=<taj_token>

Trebalo bi da vidiš poruku o uspehu, a na Lead-u da je Status = Unsubscribed i Email Opt Out = true (ako si ostavio tu liniju).


Ako želiš, mogu da ti dodam i logiku za postavljanje UnsubscribedDate__c ili audit (Task/Note) kad se neko odjavi.




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
