# openHAB-RuleManager
Will manage rules in openHAB.

## Items

At first you have to create an `Switch` item for each Rule you will `enable` or `disable`. And you have to create an `Switch` item for each Rule you want to run:

```
Group RuleManager "Rule Manager"

Switch exampleRule "Enable/Disable example rule" (RuleManager)
Switch isRunningExampleRule "Is example rule running?" (RuleManager) 
```

## Sitemaps

You can then add following to your sitemap:

```
Text label="Rule Manager" icon="control" {
    Switch item=exampleRule label="Enable/Disable example rule"
    Switch item=isRunningExampleRule label="Is example rule running?"
}
```

## Rules

For enabling and disabling a rule you can add following rules:

```
rule "Rule Manager example rule enable"
when
    Item exampleRule changed from OFF to ON
then
    executeCommandLine("/usr/bin/sshpass","-p","habopen","/usr/bin/ssh","-p","8101","openhab@localhost","openhab:automation","enableRule","<uid>","true")
end

rule "Rule Manager example rule disable"
when
    Item exampleRule changed from OFF to ON
then
    executeCommandLine("/usr/bin/sshpass","-p","habopen","/usr/bin/ssh","-p","8101","openhab@localhost","openhab:automation","enableRule","<uid>","false")
end
```

To check if an rule is running you can do following:

```
rule "example rule"
when
    <trigger_example_rule>
then
    isRunningExampleRule.postUpdate(ON) // execution of example rule is started
    
    <example_rule>
    ... // the actual rule
    </example_rule>
    
    isRunningExampleRule.postUpdate(OFF) // execution of example rule is finished
end
```

Is it possible to stop/abort a rule execution? Yes. You can extend the previous rule as follows:

```
rule "example rule"
when
    <trigger_example_rule>
then
    isRunningExampleRule.postUpdate(ON) // execution of example rule is started
    
    <example_rule>
    
    if (isRunningExampleRule.state == ON) {
        <part_of_example_rule>
        ... // a part of the example rule
        </part_of_example_rule>
    } else {
        <undo_previously_changes> // maybe you have to undo previously changes. If you changed an switch item to ON you can as example change it to OFF.
    }
    
    if (isRunningExampleRule.state == ON) {
        <another_part_of_example_rule>
        ... // a another part of the example rule
        </another_part_of_example_rule>
    } else {
        <undo_previously_changes> // maybe you have to undo previously changes. If you changed an switch item to ON you can as example change it to OFF.
    }
    
    </example_rule>
    
    isRunningExampleRule.postUpdate(OFF) // execution of example rule is finished
end
```

If you structured your rule execution into multiple rules which are executed one after the other you can change this rules to as example:

```
rule "part of example rule"
when
    <trigger_by_example_rule>
then
    if (isRunningExampleRule.state == ON) {
        <part_of_example_rule>
        ... // a part of the example rule
        </part_of_example_rule>
     else {
        <undo_previously_changes> // maybe you have to undo previously changes. If you changed an switch item to ON you can as example change it to OFF.
    }
end
```

So and how can I stop this rule (`example rule`) from execution? You have to set the state of the switch `isRunningExampleRule` to `OFF`. During execution, this switch should always be `ON`. However, this switch is only set to `ON` at the very beginning when the rule is started. If you set this switch to `OFF`, the rule will not be executed further by the `if`-statements. This happens because during the execution of the rule this switch is not set to `ON` again. The rule runs through and does not execute anything more. At the end, this switch would theoretically be set to `OFF` again. However, this then remains without effect.
