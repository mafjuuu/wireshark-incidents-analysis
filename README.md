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

Because the website does not use HTTPS, it does not encrypt anything. By expanding the packet details of that POST request, I easily found my own test credentials written in plaintext inside the data fields, requiring absolutely no decryption.

<img width="268" height="372" alt="image" src="https://github.com/user-attachments/assets/dd3ef74e-8ce0-4e60-80df-c87857d4d367" />




