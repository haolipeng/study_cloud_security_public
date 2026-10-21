解读如下规则：

```
alert tcp $EXTERNAL_NET any -> $HOME_NET 21 (msg:"ET FTP Possible FTP Daemon Username DELETE FROM SQL Injection Attempt"; flow:established,to_server; content:"USER"; depth:4; content:"DELETE"; within:200; nocase; content:"FROM"; distance:0; nocase; pcre:"/DELETE.+FROM/i"; reference:url,en.wikipedia.org/wiki/SQL_injection; classtype:attempted-user; sid:2009982; rev:2; metadata:affected_product Web_Server_Applications, attack_target Web_Server, created_at 2010_07_30, deployment Datacenter, confidence Medium, signature_severity Major, tag SQL_Injection, updated_at 2019_07_26, mitre_tactic_id TA0001, mitre_tactic_name Initial_Access, mitre_technique_id T1190, mitre_technique_name Exploit_Public_Facing_Application;)
```

