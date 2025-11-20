<h1>Twomillion-Writeup-HTB</h1>
This write-up covers my full exploitation of the 2million HTB machine, starting from basic enumeration and digging through obfuscated JavaScript, to exploiting a flawed API with Burp Suite for admin access. From there, I move into command injection, credential discovery via .env, SSH access, and finally rooting the box using an OverlayFS vulnerability. It’s a quick but detailed walkthrough of the exact steps and thinking that led to the compromise.<br><br>
<img width="940" height="768" alt="image" src="https://github.com/user-attachments/assets/cb3c2947-816b-44c6-b57d-a53656756ce3" /><br><br>
Tools Used:<br>
•	Nmap<br>
•	BurpSuite<br><br>

I started with a NMAP Scan to detect any open ports:<br><br>
<img width="939" height="320" alt="image" src="https://github.com/user-attachments/assets/a33be440-64a9-4f6b-98b6-588edc0b04c9" /><br><br>

The page wouldn’t load. A quick DNS resolution solved it:<br>
*echo”10.10.11.221 2million.htb” | sudo tee -a /etc/hosts*<br>
Once the page opened, I examined the site and found a suspicious link.<br><br>
<img width="940" height="756" alt="image" src="https://github.com/user-attachments/assets/13437205-b4f4-4428-98c0-fbb0ff2a8a47" /><br><br>

2million.htb/invite:<br><br>
<img width="961" height="476" alt="image" src="https://github.com/user-attachments/assets/9e970db9-b240-4afe-ba88-195f87b614e7" /><br><br>

Checking the source code, something interesting popped up inside a JavaScript file:<br><br>
<img width="939" height="514" alt="image" src="https://github.com/user-attachments/assets/f3776007-7b56-4994-b47d-1595eb63f71f" /><br><br>

The script looked obfuscated, one giant unreadable line. So I grabbed it from js/inviteapi.min.js.I copied the code into ChatGPT and asked for a deobfuscated version, which thankfully came back clean enough to understand.<br><br>
<img width="694" height="949" alt="image" src="https://github.com/user-attachments/assets/122ec624-2ff5-4f72-9f4f-8f0c42757c77" /><br><br>
<img width="723" height="647" alt="image" src="https://github.com/user-attachments/assets/2955a612-8ffd-4265-865c-019b1582d695" /><br><br>

I executed a curl command to get this into my machine:<br><br>
<img width="940" height="219" alt="image" src="https://github.com/user-attachments/assets/d6bffd5c-befe-4986-8b1f-123168f69be0" /><br><br>

Then, I decrypted the code using base64 command:<br>
	echo ‘UzhKVzAtOEpYWk0tQzlER1QtS1VXUEw=’ | base64 -d<br><br>
<img width="748" height="106" alt="image" src="https://github.com/user-attachments/assets/e0284a13-45d2-4d28-95fe-976ffc89d125" /><br><br>

I pasted the decoded part into the invite page, logged in using fake credentials, and landed on the home page. After exploring the site, I eventually found an “Access” page where VPN keys were generated.<br><br>
<img width="939" height="373" alt="image" src="https://github.com/user-attachments/assets/9ad27caa-3b1e-4b6a-a2fd-f045c5cd2667" /><br><br>
<img width="940" height="495" alt="image" src="https://github.com/user-attachments/assets/ddbe7a33-8339-4390-a8af-ecfc71da50b8" /><br><br>

Time to bring in Burp Suite!<br>
I intercepted the request responsible for generating the VPN, and after poking around with repeater, I noticed something big:<br><br>
<img width="940" height="253" alt="image" src="https://github.com/user-attachments/assets/5a21c284-7171-42c9-9d36-704f74093bdc" /><br><br>

So, I modified the address to v1 (version 1):<br><br>
<img width="939" height="395" alt="image" src="https://github.com/user-attachments/assets/1b99d977-d2d1-4869-81c4-8953ab15dbca" /><br><br>
<img width="940" height="256" alt="image" src="https://github.com/user-attachments/assets/a07c6b30-6809-459d-b251-59e333959e81" /><br><br>

I just need to change “is_admin”: 1 and I can enter as admin.<br>
I’ll try getting in as an admin using the PUT method I got:<br><br>
<img width="940" height="256" alt="image" src="https://github.com/user-attachments/assets/5732b3e9-0ab6-4991-a81c-d4518f5f01ac" /><br><br>
“Invalid content type”. Alright, so the issue was not the field but the format.<br><br>

Before modifying the request, I needed to figure out the correct Content-Type header it expected. In Burp Suite, I right-clicked on the request and selected “Change request method.” Doing this twice switched it over to a POST request, which conveniently revealed the Content-Type header that wasn’t visible earlier.<br>
Now, since the response shows the Content-Type as application/json, I need to match that on the request side as well. After updating the header, switch the request method back to PUT, because the server won’t accept the admin change when using POST.<br><br>
<img width="940" height="256" alt="image" src="https://github.com/user-attachments/assets/ad834777-d481-4b98-a572-a2646e70cb44" /><br><br>

“missing parameter: email”, So I’ll add the parameter and the ‘is_admin:’ 1 too:<br><br>
<img width="940" height="256" alt="image" src="https://github.com/user-attachments/assets/1c4f7b7d-ff2a-4316-9495-db966806a062" /><br><br>

Got the admin id! Now that I had admin privileges, I intercepted another VPN-related request, sent it to repeater, and verified the authentication: <br><br>
<img width="940" height="256" alt="image" src="https://github.com/user-attachments/assets/558caeeb-7e12-4290-a36a-0fb997440ea5" /><br><br>

Authenticated as admin! Now let’s check other methods of admin. The one left is generate vpn. Let’s try this:<br><br>
<img width="940" height="256" alt="image" src="https://github.com/user-attachments/assets/c26d75b0-5a42-40a0-8d77-cd4af40edc1d" /><br><br>

Let’s change that:<br><br>
<img width="938" height="403" alt="image" src="https://github.com/user-attachments/assets/288082e6-d8a5-4088-bbc5-1826ce344c55" /><br><br>

Didn’t return anything useful. So, let’s try command injection. Since we already had an ID parameter, I replaced it with something like:<br><br>
<img width="949" height="238" alt="image" src="https://github.com/user-attachments/assets/e846a8f0-e334-4bf9-a07e-9b283e270437" /><br><br>
Yup it worked!<br>
I now had command execution on the machine. <br>
**ls** shows the list of files here.<br>
**Ls -la** contained .env file so let’s check that out:<br><br>
<img width="939" height="253" alt="image" src="https://github.com/user-attachments/assets/4e83fdba-6fba-44ee-8584-d8d6db6407ec" /><br><br>
I found a credential sitting there in plain text.<br>
Now we can do ssh and get into the machine as admin:<br><br>
	**ssh admin@2million.htb** or **ssh admin@10.10.11.221**<br>
Password: **SuperDuperPass123**<br><br>

<img width="606" height="178" alt="image" src="https://github.com/user-attachments/assets/3ac0a180-3b53-4355-8685-469067658b40" /><br><br>
LET’S GOOO!!!<br>
I tried running linpeas, but the machine didn’t support it. So let’s try another way:<br><br>
<img width="940" height="864" alt="image" src="https://github.com/user-attachments/assets/77d65053-9268-43a9-80dc-2bfb36f269df" /><br><br>
<img width="939" height="678" alt="image" src="https://github.com/user-attachments/assets/494e35c7-27c6-438d-a8dd-a5f48a9dfdfa" /><br><br>
According to the mail, it got something to do with overlay-cve. A quick search brought me to a GitHub PoC:<br>
*https://github.com/DataDog/security-labs-pocs/blob/main/proof-of-concept-exploits/overlayfs-cve-2023-0386/poc.c*<br>
copied that and saved it as poc.c<br><br>
<img width="938" height="436" alt="image" src="https://github.com/user-attachments/assets/45084ffb-73ec-46f3-8d72-ea10e237a9f4" /><br><br>
<img width="713" height="211" alt="image" src="https://github.com/user-attachments/assets/16eecf1e-3975-4a73-bf50-ec1cbc45bdcc" /><br><br>
So this was 2million!
