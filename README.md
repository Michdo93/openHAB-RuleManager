# openHAB-RuleManager
Will manage rules in openHAB.

## Items

At first you have to create an `Switch` item for each Rule you will `enable` or `disable`:

```
Group RuleManager "Rule Manager"

Switch exampleRule "Enable/Disable example rule" (RuleManager)
```

## Sitemaps

You can then add following to your sitemap:

```
Text label="Rule Manager" icon="control" {
    Switch item=exampleRule label="Enable/Disable example rule"
}
```

## Rules

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
