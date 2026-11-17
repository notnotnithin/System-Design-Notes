---
hovernotes-transcript-of: doc_bb965ebb-605e-400d-9a62-1048d3bc5385
hovernotes-transcript-version: 2
note: "[[02-Network-Protocols]]"
source: "https://www.youtube.com/watch?v=JwTiZ9ENquI&list=PL6W8uoQQ2c63W58rpNFDwdrBnq5G3EfT7&index=2"
updated: 2026-11-16T10:06:13.957Z
---

# 02-Network-Protocols — Transcript

**0:00 → 0:41**

Hello Hello Hey guys, welcome to Concept and Coding Today we are starting our high level design video which is known as network protocols and as planned, we have covered network protocols clients of a model, peer to peer model, TCP, UDP when should we use? Web software, WebRTC if we have to design whatsapp which protocol should we use? If we have to create something like google meet which protocol should we use? So let's start.

**0:41 → 1:25**

नेटवर्क प्रोटोकॉल कोई यहां पर देखोगे कि जब भी हम दो सिस्टम ओर दो अपस में अपस में एक्षियंग समंदिभर आपस में जो नेटवर्क जो है त्रीखिया नेटवर्क यो जो है यह क्या rules and regulations define करता है so network protocol defines the rules so that two systems can communicate with each other दोनों अगर same language जानते होंगे तभी बात कर पाएंगे so that's where the network protocol helps तो यहां पर वो कुछ rules define करता है कि आपको message बेजना है तो इस तरीके से बेजे तो आपने college में बढ़ होगा कि यहां पर हमारे पास seven kind of network 7 लेर सोथ ये इक इस लेर साइट

**1:25 → 2:11**

application layer, presentation layer, session layer, transport layer, network layer, data link layer kind of this thing. So we will not cover all of these, we will cover two important layers, application layer and transport layer. Let's start with the application layer first. Application layer, I will divide the two parts. First, client server, protocols. And I will do peer to peer protocols.

**2:11 → 2:56**

Okay, in client server, we have a protocol in our client. STTP, FTP, SMTP, WebSockets. These are all client server protocols. Peer to peer is WebRTC. So here the most important thing is STTP, WebSockets, WebRTC. These are the most important and use the high level design. So quickly, let's see what is client server and peer to peer. In client server, we have to create some space.

**2:56 → 3:41**

Let's take client server What happens in client server? We have one web browser Which is considered as a client And we have another system called web server And our apps are running, which we call server So what happens in client server model? Client makes a request And server gives a response back It's a one way communication That client always initiate talk with server

**3:41 → 4:26**

and then you will respond so you can take HTTP, FTP, SMTP but the web socket is different so how is it different? because there is a client and server so here is a client and server but web socket is a bi-directional communication so client can talk to server and server can talk to client that is the only difference but is it peer to peer? we have mistakes so if we have server but this is client 1 and here is my client 1

**4:26 → 5:11**

client 2 client can talk to server server can talk to client 1 client can talk to server server can talk to client 2 but this will not be a deal so it's not a peer to peer and when we have like kind of whatsapp design telegram design so when we have a messaging app then we will use web socket because if we have a message then we have to send client because it will be inefficient so when we have a message

**5:11 → 5:57**

So that's where the web socket comes into the picture bi-directional talk So here we can quickly see what you mean by HTTP Hypertext Transfer Protocol So it can be used So this is connection oriented So connection oriented Here connection creates Here one connection creates And here we access web pages Here we jump from one page to another And what is here? File Transfer Protocol File Transfer 2 connections One is Control Connection The other is Data Connection Data Connection can be created and disconnected

**5:57 → 6:42**

This is the pro connection always made in FTP So this is the difference SMTP is always used with IMAP and POP So SMTP is used for sending the email Sending the email And IMAP is used for accessing the email Accessing and reading the email The POP3 is not used What happens when the email is read and download It will delete the email

**6:42 → 7:27**

So we can read from multiple computers That's why SMTP with IMAP is widely used Now let's see, it's also a client server Here is MTA, Message Transfer Agent It's a client, so it's a user User send messages, here is user agent User send messages to MTA client MTA client talks to MTA server Message Transfer agent server Message Transfer agent passes to the user agent again And here is a mail deliver Here is a mail send, mail receive So this way we can see, here is client and server Yeah.

**7:27 → 8:12**

So we will not go much deep into SMTP But what I want to tell you is that client server You should understand that client server and peer to peer What difference is the difference in the web socket What is the client server model? Now let's see what is peer to peer We have seen the web socket What happens here is our server Here is our client Here is my client They all can talk with each other All can be client and server All can be server So we will tell it to peer to peer And here is the web RTC is a peer to peer protocol.

**8:12 → 8:57**

जाने की जरुरत नहीं है और अप अपसे दो मशीन आपसे बात कर सकती है एक है एक यहां पर अब आपक्षप्ट आप से लिए एक और यहां पर आपक्षप्ट अब नेटवोग लेख तो अब यहां पर हमारे पाद दो हैं TCP IP एन UDP IP तो TCP IP में क्या होता है? PCP IP में आपको पता होगा कि यहां पे हम सबसे पहले एक virtual connection maintain करते हैं

**8:57 → 9:42**

That virtual connection through data What happens here? We make packets Data packets And they are giving sequences Now that data packets are going That we have made connections So here on server side, there is a sequence But there is sequencing So here in TCP IP, ordering is maintained And if there is no data packet So what is it? And every packet is acknowledged Acknowledge for 1 Acknowledge for 2 If there is not a TCP Acknowledge for 2 Then it will send again So TCP IP What happens here?

**9:42 → 10:27**

It is ordered, it is divided, it is acknowledged, it is maintained Now UTP, what happens in UTP? That the data I have to send, There is no connection maintained here There are various connections, And it divides it in the data-grams, And it is sent here, parallelly, on the multiple virtual connections There is no connection here So it is possible that the first packet is not maintained Because it is not maintained here, So if it is not done in the first packet, If it is not done in the first packet, If it is not done in the first packet, Then it is possible,

**10:27 → 11:13**

But UDP is the main thing that it's fast It's fast And why fast you are getting to know? Because we need to acknowledge Connection, maintain Ordering, maintain We send it, it's just the best effort It's just try its best Mechanism to transport the data And when UDP comes here? When we live streaming When we call video calling When I call video calling, I'm thinking that something missed me But you will listen to the previous video So the previous video will not listen to the previous video

**11:13 → 11:58**

What are the use cases where UDP comes into the picture? And this web app RTC is here UDP use. In the lower layer. So this is the protocol. Web RTC is the protocol. Which uses the UDP to transfer the data. And because it is fast. Because I am live stream, I need to have data fast access. So where I have the requirement to transfer data, I will use UDP, where I have ordering is not an issue. Data packet is lost, so it will be okay. And when do you use the web socket? When we have a messaging app design, we will use the web socket. Why do I have two way communication?

**11:58 → 12:43**

Because if I have user 1, user 2 has message to user 1, client 1. So, server 1 has a message. Now, server 2 has to talk to client 1. So, server 2 client 2 has to talk to client 2. So, it's a 2 way communication. So, web socket server will use 2 way communication. In case of messaging. Based upon the requirement, I will use HTTP and FTP. FTP will never use it. Because FTP is not encrypted. So, here is client. Here is server. Here is control connection.

**12:43 → 13:28**

So, whatever file is transferred is not encrypted. That is why FTP is not secure. That is why we are not using it. That is why we are using HTTPS secure transfer protocol. Right? Hypertext transfer protocols are secure. So, we are using HTTPS. That's why. So guys, overall, what is network protocols? What is the app, client server, peer-to-peer? And what we will use in the use cases. Mostly, HTTP will be used. When we design a messaging app, then we will use web software. When we use live streaming, video, or video calling, then we will use UTP or WebRTC. And when will TCP be used?
