|User Type|Credentials Stored|Where Authentication Events Appear|
|---|---|---|
|Domain user|NTDS.dit (on )|Domain Controller|
|Local user|SAM (on local machine)|Local machine only|\
## Kerberos Authentication

![[Pasted image 20260702114356.png]]

| Step | What Happens                         | Event ID | Where Logged      |
| ---- | ------------------------------------ | -------- | ----------------- |
| 1    | User requests a TGT                  | 4768     | Domain Controller |
| 2    | User requests a TGS                  | 4769     | Domain Controller |
| 3    | User creates a session on the target | 4624     | Target server     |
![[Pasted image 20260702114419.png]]
Failed : 
![[Pasted image 20260702114437.png|572]]
## Encryption Types in 4768/4769

Tickets usually are encrypted, and AD uses two encryption types for this:

- **RC4** encryption appears in environments with older systems or applications that don't support AES.
- And **AES-256** encryption, which modern systems use.

Understanding what should usually appear in our environment helps us determine which encryption types to expect and where they come from.

|Value|Algorithm|When You See It|
|---|---|---|
|0x12|AES-256|Modern systems, Windows 2008+ domain functional level|
|0x17|RC4-HMAC|Legacy systems, older applications, cross-forest trusts|
![[Pasted image 20260702114515.png|511]]
Let's say we want to filter for TGT requests in Splunk. We can use the following query:
```sql
index=* EventCode=4768
| table _time, Account_Name, Client_Address, Ticket_Encryption_Type
```

This shows us:

- Who tried to authenticate ( `Account_Name`),
- When ( `_time`),
- From which machine ( `Client_Address`),
- And the encryption type for the ticket requested ( `Ticket_Encryption_Type`).

## NTLM Authentication
NTLM is a legacy authentication protocol used when Kerberos is unavailable. This happens when accessing resources by IP address, when the target system cannot be found in DNS, or during authentication to non-domain systems.

|Step|What Happens|Event ID|Where Logged|
|---|---|---|---|
|1|The target server asks the DC to validate credentials|4776|Domain Controller|
|2|Session created on target|4624|Target server|

![[Pasted image 20260702114804.png]]
Event 4776 appears on the Domain Controller when NTLM credentials are validated. Common scenarios include:

- Accessing file shares by IP address ( `\\10.0.1.50\Shared`)
- Legacy applications that don't support kerberos
- Authentication across untrusted domains

To view NTLM authentication attempts on the DC in Splunk:
```sql
index=* EventCode=4776
| table _time, Logon_Account, Source_Workstation
```
![[Pasted image 20260702114926.png]]
This shows us:

- Which account was authenticated ( `Logon_Account`),
- From which machine ( `Source_Workstation`).

Meanwhile, on the target host, it appears as follows:
```sql
index=* EventCode=4624 Account_Name=michelle.smith Authentication_Package=NTLM
| table _time host user Workstation_Name Source_Network_Address Authentication_Package
```

![Splunk output for NTLM logon on target host](https://tryhackme-images.s3.amazonaws.com/user-uploads/68d2c1e7ab94268f6271de1d/room-content/68d2c1e7ab94268f6271de1d-1770130895298.png)
