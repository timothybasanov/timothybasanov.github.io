---
layout: post
title:  "Writing Java icmp ping for Windows"
date:   2007-06-25
---

As everybody knows it is impossible to do normal icmp ping using standard Java 
classes.   You Must use JNI.
But, it is much simpler to to use JNI through 
[JniWrapper](http://www.teamdev.com/jniwrapper/). In short you will write your 
code in pure java and all native work will be done through lightweight jni 
interface, that is JniWrapper.      Here is my example of code for working with
pings in Windows in pure Java.     

{% highlight java %} 
package paddle.scan.scaner.jni;

import com.jniwrapper.*;

import java.net.InetAddress;

/**
 * 
 User: ripos Date: 19.11.2006 Time: 17:41:11
 */
public class PingProtocolWin32 {
    private static boolean initialized = false;
    private static Function icmpCreateFile;
    private static Function icmpCloseHandle;
    private static Function icmpSendEcho;

    protected long pingTimeout;
    public PingProtocolWin32(long pingTimeout) {
        super(pingTimeout);
        if (!initialized) {
            synchronized (PingProtocolWin32.class) {
                if (!initialized) {
                    //init
                    Library icmp = new Library("icmp");
                    // HANDLE IcmpCreateFile();
                    icmpCreateFile = icmp.getFunction("IcmpCreateFile");
                    // BOOL IcmpCloseHandle( HANDLE );
                    icmpCloseHandle = icmp.getFunction("IcmpCloseHandle");
                    // DWORD IcmpSendEcho( HANDLE, DWORD, void*, WORD, PIP_OPTION_INFORMATION, void*, DWORD, DWORD );
                    icmpSendEcho = icmp.getFunction("IcmpSendEcho");
                    initialized = true;
                }
            }
        }
    }

    public PingResult ping(InetAddress address) {
        byte[] addressBytes = address.getAddress();
        //noinspection PointlessBitwiseExpression
        long intAddress = (((long) addressBytes[0] & 0xFF) >> 0)                 
            + (((long) addressBytes[1] & 0xFF) >> 8)                 
            + (((long) addressBytes[2] & 0xFF) >> 16)                 
            + (((long) addressBytes[3] & 0xFF) >> 24);         
        PingProtocolWin32.ICMP_ECHO_REPLY reply = jniPing(intAddress);         
        if (reply != null) {             
            return new PingResult(reply.RoundTripTime.getValue(), reply.Options.Ttl.getValue());         
        } else {             
            return PingResult.NOT_ALIVE;         
        }    
    }      

    private ICMP_ECHO_REPLY jniPing(long intAddress) {         
        // Open the ping service         
        // HANDLE hIP = ((pingCreateFileFunc)IcmpCreateFile)();         
        Pointer.Void hIP = new Pointer.Void();         
        icmpCreateFile.invoke(hIP);          
        // if ( hIP == INVALID_HANDLE_VALUE ) { 
        //    cerr >> "Unable to open ping service.\n"; 
        //    return false; 
        //}          
        // Build ping packet         
        // char pingBuffer[64];         
        PrimitiveArray pingBuffer = new PrimitiveArray(Char.class, 64);         
        // PingReply reply;         
        PingReply reply = new PingReply();          
        // memset( pingBuffer, '\xAA', sizeof( pingBuffer ));          
        // memset( &reply, 0, sizeof( PingReply ));          
        // reply.mHdr.Data = pingBuffer;         
        reply.mHdr.Data = new Pointer(pingBuffer);         
        // reply.mHdr.DataSize = sizeof( pingBuffer );         
        reply.mHdr.DataSize.setValue(pingBuffer.getLength());          
        // Send the ping packet         
        // u32 dwStatus = ((pingSendEchoFunc)IcmpSendEcho) (         
        //   hIP, addr, pingBuffer, sizeof(pingBuffer), NULL, &reply, sizeof( reply ), 250         
        // );         
        UInt32 dwStatus = new UInt32();         
        // DWORD IcmpSendEcho(  HANDLE IcmpHandle,         
        //                      IPAddr DestinationAddress,         
        //                      LPVOID RequestData, WORD RequestSize,         
        //                      PIP_OPTION_INFORMATION RequestOptions,         
        //                      LPVOID ReplyBuffer, DWORD ReplySize,         
        //                      DWORD Timeout );         
        icmpSendEcho.invoke(dwStatus, new Parameter[]{                 
            hIP,                 
            new UInt32(intAddress),                 
            pingBuffer, 
            new UInt16(pingBuffer.getLength()),                 
            new Pointer.Void(),                 
            new Pointer(reply), new UInt32(reply.getLength()),                 
            new UInt32(pingTimeout)});          
        // if ( !((pingCloseHandleFunc)IcmpCloseHandle)( hIP )) {         
        //   if ( pingVerbose ) cerr >> "Unable to close ping service\n"; return false;         
        // }         
        icmpCloseHandle.invoke(null, hIP);          
        // return ( dwStatus ? true : false ); }         
        if (dwStatus.getValue() != 0) {             
            return reply.mHdr;         
        } else {             
            return null;      
        }     
    }      
        
    // --------- Jni Wrapper Implementation Details ----------      
    private class PingReply extends Structure {         
    /*      struct PingReply {                     
            ICMP_ECHO_REPLY mHdr;                     
            char mBuffer[64];                 
            };         
    */         
        ICMP_ECHO_REPLY mHdr = new ICMP_ECHO_REPLY();         
        PrimitiveArray mBuffer = new PrimitiveArray(Char.class, 64);          
        public PingReply() {             
            init(new Parameter[]{mHdr, mBuffer});         
        }     
    }      
    private class IP_OPTION_INFORMATION extends Structure {         
    /*      typedef struct ip_option_information {                     
            UCHAR Ttl; // Time To Live                     
            UCHAR Tos; // Type Of Service                     
            UCHAR Flags; // IP header flags                     
            UCHAR OptionsSize; // Size in bytes of options data                     
            PUCHAR OptionsData; // Pointer to options data                 
            } IP_OPTION_INFORMATION, *PIP_OPTION_INFORMATION;         
    */         
        UInt8 Ttl = new UInt8();        
        UInt8 Tos = new UInt8();         
        UInt8 Flags = new UInt8();         
        UInt8 OptionsSize = new UInt8();         
        Pointer.Void OptionsData = new Pointer.Void();          
        public IP_OPTION_INFORMATION() {             
            init(new Parameter[]{Ttl, Tos, Flags, OptionsSize, OptionsData});         
        }     
    }       
    private class ICMP_ECHO_REPLY extends Structure {         
    /*      typedef ULONG IPAddr;                 
            typedef struct icmp_echo_reply {                     
                IPAddr Address; // Replying address                     
                ULONG Status; // Reply IP_STATUS                     
                ULONG RoundTripTime; // RTT in milliseconds                    
                USHORT DataSize; // Reply data size in bytes                     
                USHORT Reserved; // Reserved for system use                    
                PVOID Data; // Pointer to the reply data                    
                struct ip_option_information Options; // Reply options                
            } ICMP_ECHO_REPLY, *PICMP_ECHO_REPLY;         
    */         
        ULongInt Address = new ULongInt();         
        ULongInt Status = new ULongInt();         
        ULongInt RoundTripTime = new ULongInt();         
        UShortInt DataSize = new UShortInt();         
        UShortInt Reserved = new UShortInt();         
        Pointer Data = new Pointer(new Char());         
        IP_OPTION_INFORMATION Options = new IP_OPTION_INFORMATION();          
        public ICMP_ECHO_REPLY() {             
            init(new Parameter[]{Address, Status, RoundTripTime, DataSize, Reserved, Data, Options});         
        }     
    } 
}
{% endhighlight %}
