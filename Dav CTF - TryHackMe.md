I began with an Nmap scan, which revealed a single open port:

Port 80 (HTTP)

<img width="766" height="219" alt="nmap scan" src="https://github.com/user-attachments/assets/a9e34507-2073-4a40-9c38-66230762575b" />

Navigating to the web server in a browser led to a default Apache page, indicating no obvious attack surface from the landing page.

To enumerate hidden directories, I used Gobuster:

<img width="735" height="387" alt="gobuster" src="https://github.com/user-attachments/assets/b17d2143-567e-4c31-9b9a-5171d563c41c" />

Gobuster revealed a directory named /webdav, which looked promising as a potential entry point.

Browsing to /webdav prompted a username and password authentication popup, suggesting the directory was protected.

<img width="1918" height="653" alt="webdav prompt" src="https://github.com/user-attachments/assets/2c1335c4-a019-415d-be0b-8ac2b40d37f7" />

After researching common WebDAV default credentials, I attempted:

wamp:xampp

These credentials worked successfully.

Once authenticated, I was presented with a directory listing containing a file named password.dav.

<img width="527" height="170" alt="webdav listing" src="https://github.com/user-attachments/assets/ebf366b8-2c76-4b4c-8f2f-36a8a2de0817" />
<img width="1918" height="653" alt="webdav prompt" src="https://github.com/user-attachments/assets/a0dd4f7b-4ae3-43be-a16e-820441deba3e" />

I attempted to crack the hash using John the Ripper, but it did not yield any results.

I then researched tools for interacting directly with WebDAV services and discovered Cadaver, a command-line WebDAV client.

I connected to the WebDAV service using:

cadaver http://<IP>/webdav

After authenticating, I used the put command to upload a PHP reverse shell from my local machine to the WebDAV directory.

<img width="615" height="202" alt="cadever" src="https://github.com/user-attachments/assets/0e417f1f-35f4-43db-8051-58b2bb2d1449" />

I set up a Netcat listener on my machine and executed the uploaded PHP reverse shell via the browser.

This successfully returned a reverse shell on the target system.

After exploring the filesystem, I located the user flag at:

/home/merlin

To identify privilege escalation vectors, I transferred LinPEAS to the target using wget and executed it.

<img width="951" height="136" alt="linpeas output" src="https://github.com/user-attachments/assets/a5232389-190c-45a8-8ad8-0439de9860c8" />

Reviewing the output revealed the following sudo rule:

(ALL) NOPASSWD: /bin/cat

In hindsight, running sudo -l would have revealed the same information (I forgot at the time 😅), but LinPEAS made it very clear.

Since /bin/cat could be run as root without a password, I used:

sudo /bin/cat /root/root.txt

This allowed me to retrieve the root flag, completing the machine.
