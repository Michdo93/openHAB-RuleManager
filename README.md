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
    isRunningExampleRule.sendCommand(ON) // execution of example rule is started
    
    <example_rule>
    ... // the actual rule
    </example_rule>
    
    isRunningExampleRule.sendCommand(OFF) // execution of example rule is finished
end
```

Is it possible to stop/abort a rule execution? Yes. You can extend the previous rule as follows:

```
rule "example rule"
when
    <trigger_example_rule>
then
    isRunningExampleRule.sendCommand(ON) // execution of example rule is started
    
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
    
    isRunningExampleRule.sendCommand(OFF) // execution of example rule is finished
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
    } else {
        <undo_previously_changes> // maybe you have to undo previously changes. If you changed an switch item to ON you can as example change it to OFF.
    }
end
```

So and how can I stop this rule (`example rule`) from execution? You have to set the state of the switch `isRunningExampleRule` to `OFF`. During execution, this switch should always be `ON`. However, this switch is only set to `ON` at the very beginning when the rule is started. If you set this switch to `OFF`, the rule will not be executed further by the `if`-statements. This happens because during the execution of the rule this switch is not set to `ON` again. The rule runs through and does not execute anything more. At the end, this switch would theoretically be set to `OFF` again. However, this then remains without effect.

There is another borderline case to consider: What if one rule is running and this is supposed to cause another rule not to be executed because in this one perhaps a trigger is used which is triggered by the other rule and these would interfere? This can also be queried and prevented. It is only fair to say that you would then not have set the triggers well and would have had to add further conditions:

```
rule "second example rule"
when
    <trigger_second_example_rule>
then
    if (isRunningExampleRule.state == OFF) {
        <second_example_rule>
        ... // the actual second example rule
        </second_example_rule>
    }
end
```

And who has understood the concept presented here correctly, also introduces to this and each further rule a switch that can enable or disable the execution of the second rule and also a switch that enables or disables the rule. If a rule is disabled completely, then it does not interfere with the execution of the other rule right from the start.

So do I have to check if this other rule is enabled or disabled?

```
rule "second example rule"
when
    <trigger_second_example_rule>
then
    if (exampleRule.state == OFF) {
        if (isRunningExampleRule.state == OFF) {
            <second_example_rule>
            ... // the actual second example rule
            </second_example_rule>
        }
    }
end
```

Of course not! The state of the switch that this rule is running should never reach `ON` by execution this rule. In this case `isRunningExampleRule` is still `OFF`.

But how can I be sure? I can switch the item to `ON` via REST API, MQTT, Karaf Console or by using my Sitemap. Well, that would not be so wise and could have fatal consequences.

One thing should be clear, I can't do the following either:

```
rule "example rule"
when
    <trigger_example_rule>
then
    if (exampleRule.state == ON) {
        isRunningExampleRule.sendCommand(ON) // execution of example rule is started
    }
    
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
    
    isRunningExampleRule.sendCommand(OFF) // execution of example rule is finished
end
```

To be clearer...

```
    if (exampleRule.state == ON) {
        isRunningExampleRule.sendCommand(ON) // execution of example rule is started
    }
```

will not work because the rule will never be triggered of the state of `exampleRule` is `OFF`.

But you can create another rule:

```
rule "check if isRunningExampleRule is actual running"
when
    Item isRunningExampleRule changed to ON
then
    if (exampleRule.state == OFF) {
        isRunningExampleRule.postUpdate(OFF)
    }
end
```

You can't outsmart the system with this anymore. Or can you? Yes you can if you make following mistake: `isRunningExampleRule.postUpdate(ON)` swap with `isRunningExampleRule.sendCommand(ON)`. 

Nevertheless, the second rule should look like this:

```
rule "second example rule"
when
    <trigger_second_example_rule>
then
    if (isRunningExampleRule.state == OFF) {
        isRunningSecondExampleRule.sendCommand(ON)
    
        <second_example_rule>
    
        if (isRunningSecondExampleRule.state == ON) {
            <part_of_second_example_rule>
            ... // a part of the second example rule
            </part_of_second_example_rule>
        } else {
            <undo_previously_changes> // maybe you have to undo previously changes. If you changed an switch item to ON you can as example change it to OFF.
        }
    
        if (isRunningSecondExampleRule.state == ON) {
            <another_part_of_second_example_rule>
            ... // a another part of the second example rule
            </another_part_of_second_example_rule>
        } else {
            <undo_previously_changes> // maybe you have to undo previously changes. If you changed an switch item to ON you can as example change it to OFF.
        }
    
        </second_example_rule>
    }
    
    isRunningSecondExampleRule.sendCommand(OFF)
end
```

## New Items

The items changed then to:

```
Group RuleManager "Rule Manager"

Switch exampleRule "Enable/Disable example rule" (RuleManager)
Switch isRunningExampleRule "Is example rule running?" (RuleManager)
Switch secondExampleRule "Enable/Disable second example rule" (RuleManager)
Switch isRunningSecondExampleRule "Is second example rule running?" (RuleManager) 
```

## New Sitemaps

The sitemap changed then to:

```
Text label="Rule Manager" icon="control" {
    Switch item=exampleRule label="Enable/Disable example rule"
    Switch item=isRunningExampleRule label="Is example rule running?"
    Switch item=secondExampleRule label="Enable/Disable second example rule"
    Switch item=isRunningSecondExampleRule label="Is second example rule running?"
}
```

## New Rules

Your rules at least look like this:

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

rule "Rule Manager second example rule enable"
when
    Item secondExampleRule changed from OFF to ON
then
    executeCommandLine("/usr/bin/sshpass","-p","habopen","/usr/bin/ssh","-p","8101","openhab@localhost","openhab:automation","enableRule","<uid>","true")
end

rule "Rule Manager second example rule disable"
when
    Item secondExampleRule changed from OFF to ON
then
    executeCommandLine("/usr/bin/sshpass","-p","habopen","/usr/bin/ssh","-p","8101","openhab@localhost","openhab:automation","enableRule","<uid>","false")
end

rule "check if isRunningExampleRule is actual running"
when
    Item isRunningExampleRule changed to ON
then
    if (exampleRule.state == OFF) {
        isRunningExampleRule.postUpdate(OFF)
    }
end

rule "check if isRunningSecondExampleRule is actual running"
when
    Item isRunningSecondExampleRule changed to ON
then
    if (secondExampleRule.state == OFF) {
        isRunningSecondExampleRule.postUpdate(OFF)
    }
end

rule "example rule"
when
    <trigger_example_rule>
then
    isRunningExampleRule.sendCommand(ON) // execution of example rule is started
    
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
    
    isRunningExampleRule.sendCommand(OFF) // execution of example rule is finished
end

rule "second example rule"
when
    <trigger_second_example_rule>
then
    if (isRunningExampleRule.state == OFF) {
        isRunningSecondExampleRule.sendCommand(ON)
    
        <second_example_rule>
    
        if (isRunningSecondExampleRule.state == ON) {
            <part_of_second_example_rule>
            ... // a part of the second example rule
            </part_of_second_example_rule>
        } else {
            <undo_previously_changes> // maybe you have to undo previously changes. If you changed an switch item to ON you can as example change it to OFF.
        }
    
        if (isRunningSecondExampleRule.state == ON) {
            <another_part_of_second_example_rule>
            ... // a another part of the second example rule
            </another_part_of_second_example_rule>
        } else {
            <undo_previously_changes> // maybe you have to undo previously changes. If you changed an switch item to ON you can as example change it to OFF.
        }
    
        </second_example_rule>
    }
    
    isRunningSecondExampleRule.sendCommand(OFF)
end
```
