<group name="local,corpnet,ad_attacks,windows,sysmon">

  <rule id="100201" level="10">
    <if_group>windows_security</if_group>
    <field name="win.system.eventID">^4769$</field>
    <field name="win.eventdata.TicketEncryptionType">^0x17$</field>
    <field name="win.eventdata.Status">^0x0$</field>
    <field name="win.eventdata.ServiceName" negate="yes" type="pcre2">(?i)krbtgt|\$$</field>
    <description>AD Possible Kerberoasting - RC4 TGS-REQ (correlates with Suricata sid:1000001)</description>
    <mitre><id>T1558.003</id></mitre>
    <group>kerberoasting,credential_access,</group>
  </rule>

  <rule id="100202" level="12" frequency="3" timeframe="60">
    <if_matched_sid>100201</if_matched_sid>
    <same_source_ip />
    <description>AD Possible Kerberoasting - Multiple RC4 TGS-REQ within 60s</description>
    <mitre><id>T1558.003</id></mitre>
    <group>kerberoasting,credential_access,</group>
  </rule>

  <rule id="100203" level="10">
    <if_group>windows_security</if_group>
    <field name="win.system.eventID">^4768$</field>
    <field name="win.eventdata.PreAuthType">^0$</field>
    <description>AD Possible AS-REP Roasting - AS-REQ without pre-authentication</description>
    <mitre><id>T1558.004</id></mitre>
    <group>asrep_roasting,credential_access,</group>
  </rule>

  <rule id="100204" level="12">
    <if_group>windows_security</if_group>
    <field name="win.system.eventID">^4662$</field>
    <field name="win.eventdata.Properties" type="pcre2">(?i)1131f6aa-9c07-11d1-f79f-00c04fc2dcd2|1131f6ad-9c07-11d1-f79f-00c04fc2dcd2</field>
    <field name="win.eventdata.SubjectUserName" negate="yes" type="pcre2">\$$</field>
    <description>AD Possible DCSync - DS-Replication-Get-Changes(-All) by non-machine account (correlates with Suricata sid:1000002)</description>
    <mitre><id>T1003.006</id></mitre>
    <group>dcsync,credential_access,</group>
  </rule>

  <rule id="100205" level="15">
    <if_group>sysmon_event1_12_13</if_group>
    <field name="data.win.eventdata.eventType" type="pcre2">(?i)SetValue</field>
    <field name="data.win.eventdata.image" type="pcre2">(?i)procdump(64)?\.exe$</field>
    <field name="data.win.eventdata.targetObject" type="pcre2">(?i)lsass</field>
    <description>Critical: Possible LSASS Credential Dumping</description>
    <mitre><id>T1003.001</id></mitre>
    <group>lsass_dump,credential_access,sysinternals,sysmon</group>
</rule>

</group>