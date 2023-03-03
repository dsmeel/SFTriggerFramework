# Salesforce Trigger Framework

## Overview

As a best practice Object should have only one trigger and triggers should be logicless. Putting logic into your triggers creates un-testable, difficult-to-maintain code. It's widely accepted that a best-practice is to move trigger logic into a handler class.

This trigger framework bundles a single **TriggerHandler** base class that you can inherit from in all of your trigger handlers. The base class includes context-specific methods that are automatically called when a trigger is executed.

The base class also provides a secondary role as a supervisor for Trigger execution. It acts like a watchdog, monitoring trigger activity and providing an api for controlling certain aspects of execution and control flow.

You can controll your triggers from metadata records like activate/deactivate and turn on/off for specific trigger action

But the most important part of this framework is that it's minimal and simple to use. 

**Deploy to Salesforce Org:**
[![Deploy](https://raw.githubusercontent.com/afawcett/githubsfdeploy/master/deploy.png)](https://githubsfdeploy.herokuapp.com/?owner=dsmeel&repo=SFTriggerFramework&ref=main)

## Usage

To create a trigger handler, you simply need to create a class that inherits from **TriggerHandler.cls**. Here is an example for creating an Account trigger handler.

```java
public class AccountTriggerHandler extends TriggerHandler {
```

In your trigger handler, to add logic to any of the trigger contexts, you only need to override them in your trigger handler. Here is how we would add logic to a `beforeUpdate` trigger.

```java
public class AccountTriggerHandler extends TriggerHandler {
  
  public override void beforeUpdate() {
    for(Account account : (List<Account>) Trigger.new) {
      // do something
    }
  }

  // add overrides for other contexts

}
```

**Note:** When referencing the Trigger statics within a class, SObjects are returned versus SObject subclasses like Opportunity, Account, etc. This means that you must cast when you reference them in your trigger handler. You could do this in your constructor if you wanted. 

```java
public class AccountTriggerHandler extends TriggerHandler {

  private Map<Id, Account> newOppMap;

  public AccountTriggerHandler() {
    this.newOppMap = (Map<Id, Account>) Trigger.newMap;
  }
  
  public override void afterUpdate() {
    //
  }

}
```

After creating handler class you need to create Tirgger_Handler__mdt record for that handler with object name and trigger actions for which you want to execute your trigger

![Trigger Handler Record Detail](/Assets/Trigger_Handler_Detail.png)

To use the trigger handler, you only need to construct an instance of your trigger handler and call the `run()` method. Here is an example of the Account trigger.

```java
trigger AccountTrigger on Account (before insert, before update, before delete, after insert, after update, after delete, after undelete) {
  new TriggerHandler().run();
}
```

## Cool Stuff

### Max Loop Count

To prevent recursion, you can set a max loop count for Trigger Handler. If this max is exceeded, and exception will be thrown. A great use case is when you want to ensure that your trigger runs once and only once within a single execution. Example:

```java
public class AccountTriggerHandler extends TriggerHandler {

  public AccountTriggerHandler() {
    this.setMaxLoopCount(1);
  }
  
  public override void afterUpdate() {
    List<Account> opps = [SELECT Id FROM Account WHERE Id IN :Trigger.newMap.keySet()];
    update opps; // this will throw after this update
  }

}
```

### Bypass API

What if you want to tell other trigger handlers to halt execution? That's easy with the bypass api:

```java
public class OpportunityTriggerHandler extends TriggerHandler {
  
  public override void afterUpdate() {
    List<Opportunity> opps = [SELECT Id, AccountId FROM Opportunity WHERE Id IN :Trigger.newMap.keySet()];
    
    Account acc = [SELECT Id, Name FROM Account WHERE Id = :opps.get(0).AccountId];

    TriggerHandler.bypass('AccountTriggerHandler');

    acc.Name = 'No Trigger';
    update acc; // won't invoke the AccountTriggerHandler

    TriggerHandler.clearBypass('AccountTriggerHandler');

    acc.Name = 'With Trigger';
    update acc; // will invoke the AccountTriggerHandler

  }

}
```

If you need to check if a handler is bypassed, use the `isBypassed` method:

```java
if (TriggerHandler.isBypassed('AccountTriggerHandler')) {
  // ... do something if the Account trigger handler is bypassed!
}
```

If you want to clear all bypasses for the transaction, simple use the `clearAllBypasses` method, as in:

```java
// ... done with bypasses!

TriggerHandler.clearAllBypasses();

// ... now handlers won't be ignored!
```

## Overridable Methods

Here are all of the methods that you can override. All of the context possibilities are supported.

* `beforeInsert()`
* `beforeUpdate()`
* `beforeDelete()`
* `afterInsert()`
* `afterUpdate()`
* `afterDelete()`
* `afterUndelete()`