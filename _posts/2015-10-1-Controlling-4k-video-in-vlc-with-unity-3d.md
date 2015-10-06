---
layout: post
title: Controlling 4k video in vlc with unity3d!
---

For a project that I am working on, I wanted to control the playback of a 4k video entirely from unity. But of course that was not going to happen. After looking at a lot of assets in the asset store that claimed to do this. I finally thought of doing something entirely diferent.

### My requirement: 
- Primary requirement: Play a 4k video with unity.
- Secondary requirement: Control the media player, playing the video.

### What I did:
- I used telnet to control vlc, I sent commands over telnet to control the video playback and I did this by opening a channel to the interface.
- In unity, I started vlc using System.Diagnostics.Process.Start("/path/to/vlc.exe")
- I setup vlc to allow for incoming telnet connections. And then I used an interface for telnet to write commands to it.

And that was it. I was able to control vlc.

### How I did it:

- ##### Vlc Setup:
    To setup vlc go to Tools > Preferences and select the Show settings > All option.
    Then set the Main Interface and Lua options like the images below.


![_config.yml]({{ site.baseurl }}/images/vlc-lua.PNG)

![_config.yml]({{ site.baseurl }}/images/vlc-main-interface.PNG)

- ##### Starting vlc from unity:

    Using just this line of code, from my Awake method, I was able to start a vlc process.
    ```
    System.Diagnostics.Process.Start("C:\\Program Files (x86)\\VideoLAN\\VLC\\vlc.exe");
    ```
        
- ##### Telnet code:
    For the Telnet code I used a Minimalist Telnet Interface that looks like this,

    ```
    using System;
    using System.Collections.Generic;
    using System.Text;
    using System.Net.Sockets;

	namespace MinimalisticTelnet
    {
        enum Verbs
    	{
        	WILL = 251,
        	WONT = 252,
        	DO = 253,
        	DONT = 254,
        	IAC = 255
    	}

    	enum Options
    	{
        	SGA = 3
    	}

    class TelnetConnection
    {
        TcpClient tcpSocket;

        int TimeOutMs = 100;

        public TelnetConnection(string Hostname, int Port)
        {
            tcpSocket = new TcpClient(Hostname, Port);

        }

        public string Login(string Username, string Password, int LoginTimeOutMs)
        {
            int oldTimeOutMs = TimeOutMs;
            TimeOutMs = LoginTimeOutMs;
            string s = Read();
            if (!s.TrimEnd().EndsWith(":"))
                throw new Exception("Failed to connect : no login prompt");
            WriteLine(Username);

            s += Read();
            if (!s.TrimEnd().EndsWith(":"))
                throw new Exception("Failed to connect : no password prompt");
            WriteLine(Password);

            s += Read();
            TimeOutMs = oldTimeOutMs;
            return s;
        }

        public void WriteLine(string cmd)
        {
            Write(cmd + "\n");
        }

        public void Write(string cmd)
        {
            if (!tcpSocket.Connected) return;
            byte[] buf = System.Text.ASCIIEncoding.ASCII.GetBytes(cmd.Replace("\0xFF", "\0xFF\0xFF"));
            tcpSocket.GetStream().Write(buf, 0, buf.Length);
        }

        public string Read()
        {
            if (!tcpSocket.Connected) return null;
            StringBuilder sb = new StringBuilder();
            do
            {
                ParseTelnet(sb);
                System.Threading.Thread.Sleep(TimeOutMs);
            } while (tcpSocket.Available > 0);
            return sb.ToString();
        }

        public bool IsConnected
        {
            get { return tcpSocket.Connected; }
        }

        void ParseTelnet(StringBuilder sb)
        {
            while (tcpSocket.Available > 0)
            {
                int input = tcpSocket.GetStream().ReadByte();
                switch (input)
                {
                    case -1:
                        break;
                    case (int)Verbs.IAC:
                        // interpret as command
                        int inputverb = tcpSocket.GetStream().ReadByte();
                        if (inputverb == -1) break;
                        switch (inputverb)
                        {
                            case (int)Verbs.IAC:
                                //literal IAC = 255 escaped, so append char 255 to string
                                sb.Append(inputverb);
                                break;
                            case (int)Verbs.DO:
                            case (int)Verbs.DONT:
                            case (int)Verbs.WILL:
                            case (int)Verbs.WONT:
                                // reply to all commands with "WONT", unless it is SGA (suppres go ahead)
                                int inputoption = tcpSocket.GetStream().ReadByte();
                                if (inputoption == -1) break;
                                tcpSocket.GetStream().WriteByte((byte)Verbs.IAC);
                                if (inputoption == (int)Options.SGA)
                                    tcpSocket.GetStream().WriteByte(inputverb == (int)Verbs.DO ? (byte)Verbs.WILL : (byte)Verbs.DO);
                                else
                                    tcpSocket.GetStream().WriteByte(inputverb == (int)Verbs.DO ? (byte)Verbs.WONT : (byte)Verbs.DONT);
                                tcpSocket.GetStream().WriteByte((byte)inputoption);
                                break;
                            default:
                                break;
                        }
                        break;
                    default:
                        sb.Append((char)input);
                        break;
                }
            }
        }
    }
    }
    ```

    And I used this in my VideoController class like this,
    
    ```
    using UnityEngine;
    using MinimalisticTelnet;
    using System;
    using System.IO;

    public class VideoController : MonoBehaviour{

    TelnetConnection tc;
    void Awake()
    {
        
        System.Diagnostics.Process.Start("/path/to/vlc");
        //create a new telnet connection to hostname "localhost" on port "4212" for vlc
        tc = new TelnetConnection("localhost", 4212);
    }

    void Start()
    {
        Debug.Log("VideoController::Started");
        
        //connect
        establishTelnet();
    }

    public void pause()
    {
        if(tc.IsConnected)
            tc.WriteLine("pause");
    }

    public void play()
    {
        if (tc.IsConnected)
        {
            tc.WriteLine("add " + Path.GetFullPath("/path/to/video/you/want/to/play"));
            tc.WriteLine("play");
        }
    }

    public void stop()
    {
        if (tc.IsConnected)
            tc.WriteLine("stop");
    }

    public void seek(int seconds)
    {
        tc.WriteLine("seek " + seconds);
    }

    void establishTelnet()
    {
        tc.Login("", "vlc", 100);
        if(tc.IsConnected)
            Debug.Log("Telnet connection:: Connection Estableshed with vlc");
    }
    void OnApplicationQuit()
    {        
        tc.WriteLine("shutdown");
        tc.WriteLine("exit");
    }
    }

    ```

And thats's pretty much it. Call your methods on video controller objects from anywhere and you can control the playback.

Here is how I am doing it,

    ```
    private VideoController vidC = GameObject.FindObjectOfType<VideoController>();
    .
    .
    .
    .
    .
    vidC.play();
    .
    .
    .
    vidC.pause();
    ```

For more commands look up the help in vlc and it should work.

Give me feedback 

[me@kumarankit.com](mailto:me@kumarankit.com)