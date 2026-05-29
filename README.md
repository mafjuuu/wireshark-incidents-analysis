# wireshark-incidents-analysis
To analyze these files safely without risking my main computer, I set up a dedicated lab environment. I created an Ubuntu Linux virtual machine (with GUI) inside VirtualBox.

I installed Wireshark on the VM to look closely at the network packets. As you can see in the screenshot below, this setup allows me to see everything happening on the network interface. The image shows a mix of live traffic, including TCP handshakes, encrypted TLSv1.3 data, and a highlighted DNS query.

Using Wireshark on Linux made it much easier to filter the noise, reconstruct HTTP streams, and find the exact indicators of compromise (IoCs) for this project.
<img width="901" height="547" alt="image" src="https://github.com/user-attachments/assets/9b4c7932-b905-4de6-92be-cac4b43f0357" />

## DNS
I used a DNS filter (dns.qry.name contains ".com") to see the communication between my device and the DNS server. This technique is useful in network forensics because it shows a history of domains the machine tried to resolve.

As seen in the screenshot, we can clearly see the Source and Destination IP addresses. Here, 10.0.2.15 is my virtual machine, and 192.168.68.1 is my gateway router acting as the DNS server.

However, it is important to note that **DNS logs do not show exactly which specific pages were visited or what the user did there**. A DNS query only reveals the main domain name. Once the IP address is found, the actual web traffic is encrypted using TLS (HTTPS). Because of this encryption, the exact content, full URLs, and transferred data remain completely hidden from network sniffing. Even with this limitation, tracking DNS requests is critical for finding suspicious domains connected to malware or C2 servers.
<img width="896" height="482" alt="image" src="https://github.com/user-attachments/assets/9438feca-aa0d-43c4-9f33-22260f8213dc" />

## HTTP Cleartext Credential Theft
Before we look at the real malware incidents, I want to show how easy it is to steal login credentials when a website uses a basic, unencrypted HTTP connection. To test this, I used a vulnerable demo banking website called Altoro Mutual. I opened the site in my lab environment, entered a test username and password, and clicked login.
<img width="938" height="408" alt="image" src="https://github.com/user-attachments/assets/2d3986b3-43f2-48e1-9f1e-7eedfe49525f" />

After capturing the network traffic, I needed to find the exact moment the login data was sent to the server. I filtered the huge mass of packets in Wireshark using the http filter and looked specifically for the HTTP POST method, which is used to submit form data.
<img width="898" height="340" alt="image" src="https://github.com/user-attachments/assets/4df42ca2-c249-4cbe-ae31-60397f1e3f9d" />

Because the website does not use HTTPS, it does not encrypt anything. To reveal these credentials, I expanded the HTML Form URL Encoded section within Wireshark's Packet Details pane, which parses the raw body of the HTTP POST request. As a result, the parameters uid=robert123 and passw=robert123 became fully visible in plaintext, demonstrating how effortlessly an attacker can intercept sensitive login data over an unencrypted connection.

<img width="268" height="372" alt="image" src="https://github.com/user-attachments/assets/dd3ef74e-8ce0-4e60-80df-c87857d4d367" />

## AgentTesla Malware Analysis (Infostealer)
After showing the basic risks of HTTP, we will now analyze a real-world infection. In this case, an employee fell victim to a phishing attack and executed a malicious file, which turned out to be the AgentTesla malware. To perform this analysis, I used real traffic data and infection artifacts from Malware-Traffic-Analysis.net. AgentTesla is a dangerous infostealer designed to silently extract sensitive data, such as browser passwords, email credentials, and keystrokes, from the infected machine.
<img width="735" height="251" alt="image" src="https://github.com/user-attachments/assets/e71d11d6-0b86-456e-add1-16b51d279a18" />

Now I will investigate the network traffic to find out exactly how the malware communicated with the attacker's server and what data was stolen.
I started by filtering the DNS traffic to see which domains the infected machine was trying to reach. As shown in the screenshot, I found several queries for ftp.ercolina-usa.com.
This is highly suspicious because ordinary office employees rarely use the FTP protocol in their daily work. In this context, it suggests that the AgentTesla malware is looking for its Command & Control (C2) server or an exfiltration point to upload stolen data.
<img width="902" height="49" alt="image" src="https://github.com/user-attachments/assets/de6ddde1-afd7-439a-8680-416a54c400a0" />

After finding the suspicious DNS requests, I applied the ftp filter to check what was actually happening during the connection. The results in the screenshot gave me a clear view of the data theft.
<img width="902" height="218" alt="image" src="https://github.com/user-attachments/assets/27b89154-f484-474a-ba70-50b11c79ac32" />

We can see the malware automatically authenticating on the server (ftp.ercolina-usa.com) with the username `ben@ercolina-usa.com` and a plaintext password. Right after logging in, the infection signature becomes obvious. The malware uses the STOR command to upload a file named PW_gary.strickman-DESKTOP..., which is a clear sign that AgentTesla collected a list of passwords belonging to the user "Gary" and sent them out. Finally, the server responds with a 226 code, confirming the file was successfully transferred. This completely confirms the data exfiltration phase of the attack.

To get a clearer view of the entire interaction, I used Wireshark's Follow TCP Stream feature. This function reconstructs the conversation, showing the client's requests in red and the server's responses in blue, making the raw network logs much easier to read.
<img width="748" height="617" alt="image" src="https://github.com/user-attachments/assets/1376587c-c944-4312-a331-bd218b4dfe86" />

As shown in the first stream snapshot, we can read the exact sequence of the breach. The infected host connects to ftp.ercolina-usa.com, passes the credentials in cleartext, and issues the first STOR command to upload Gary's system passwords. 
<img width="747" height="394" alt="image" src="https://github.com/user-attachments/assets/1a2ae70d-6d7b-436d-9a9c-e8acf2f449c2" />

Looking further into the stream, the malware didn't stop there. It performed a thorough exfiltration by uploading targeted credential stores from local web browsers. We can see separate STOR commands for Chrome (CO_Chrome_Default.txt...) and Microsoft Edge (CO_Edge Chromium_Default.txt...) databases. This clear visualization serves as definitive proof of how an infostealer operates once it gains a foothold on a system.

Beyond looking at the network traffic, we can investigate the initial access vector. The source files include the raw phishing email that started the entire infection chain. By opening the .eml file, we can look directly at the email headers and the HTML body code to understand how the attacker tricked the employee.

<img width="746" height="405" alt="image" src="https://github.com/user-attachments/assets/58cf2dba-405e-45c7-b000-9c8892afa399" />

As shown in the headers, the email arrived with the subject line PURCHASE QUOTATION. The attacker is using impersonation techniques here. The From: header shows that the message pretends to come from a real company employee (`sertan@acronas.com.tr`). This is a classic social engineering trick designed to create urgency and make the business target open the malicious attachment.
<img width="736" height="425" alt="image" src="https://github.com/user-attachments/assets/c93c3a58-1807-4e99-a993-0283a7e720cf" />


Looking at the HTML body source code, we can read the actual message text sent to the victim. The email reads: "We are Turkish company. Please see technical specifications in the attached file." The attacker signs off as a "Purchase Manager" and includes real corporate details, phone numbers, and website links to build trust. The "attached file" mentioned in the text contained the AgentTesla payload, which executed as soon as the user fell for the trick and opened it. 
After confirming the infection, as a SOC Analyst, I would immediately isolate the infected host from the network to stop any further data theft. I would also force a password reset for all compromised user accounts and block the malicious FTP domain on the company's firewall. Finally, I would document all gathered Indicators of Compromise (IoCs) to help the incident response team conduct a deeper forensic investigation.

## SmartApeSG RAT malware Analysis
After investigating the infostealer, I moved to a more advanced threat: a Remote Access Trojan (RAT) known as SmartApeSG. While an infostealer just grabs data and leaves, a RAT stays on the system to give the attacker full, long-term control over the infected machine. 
<img width="897" height="236" alt="image" src="https://github.com/user-attachments/assets/f5ccc8a9-390c-48b6-bfb3-d1119ede8dcc" />

In this analysis, I focused on identifying the C2 (Command & Control) communication. By looking at the network logs, I can show how the malware persistently "beacons" out to the attacker's server, waiting for remote instructions. This type of malware is especially dangerous because it allows for real-time spying and further exploitation of the internal network. To investigate the SmartApeSG RAT, I applied the http filter to analyze the web traffic originating from the infected host (10.12.17.101). The network logs quickly revealed a highly automated pattern, commonly known as a heartbeat or beaconing.
<img width="743" height="196" alt="image" src="https://github.com/user-attachments/assets/a543905c-79bc-49ec-bf59-87ccd22f0b42" />

As shown in the screenshot, the compromised machine continuously sends HTTP POST requests to an external IP address (194.180.191.64) targeting a suspicious path /fakeurl.htm. By looking closely at the timestamps, the requests occur in flawless 60-second intervals. This absolute precision proves that the traffic is generated by a script or malware check-in mechanism rather than human web browsing, confirming the active connection to a Command & Control (C2) server. To uncover the exact commands being exchanged, I used the Follow TCP Stream feature to reconstruct the HTTP payloads. The raw stream reveals how the SmartApeSG RAT manages its remote session under the guise of legitimate software.

<img width="748" height="543" alt="image" src="https://github.com/user-attachments/assets/0b467e07-b837-4ce0-995f-44117626896a" />

As shown in the screenshot, the malware uses a highly descriptive header: User-Agent: NetSupport Manager/1.3. By using a modified version of a known remote administration tool, the attackers attempt to blend in with normal network operations. Inside the HTTP request body, the actual C2 protocol becomes visible. The infected host sends a CMD=POLL request, which acts as a check-in to see if the operator has queued any new instructions. The server (NetSupport Gateway/1.8) responds with CMD=ENCD and an encrypted data payload (DATA=...), establishing a persistent execution channel.

<img width="748" height="583" alt="image" src="https://github.com/user-attachments/assets/e149a6d3-4ec2-4ad2-b7b5-412ec269e024" />

The second stream capture shows the continuation of this loop. The client repeatedly responds with its own CMD=ENCD packets, uploading obfuscated system data or execution results back to the C2 server at 194.180.191.64. This continuous loop of encoded exchange confirms that the attacker has successful interactive control over the victim's environment.

Based on the evidence gathered, as a SOC Analyst, I would immediately isolate the infected workstation from the network to terminate the active remote control session. I would then blacklist the C2 IP address 194.180.191.64 on the corporate firewall to prevent any further communication with the malicious infrastructure. Finally, I would create a detection rule for the unauthorized NetSupport Manager User-Agent string to hunt for potential sister infections across the environment.


































