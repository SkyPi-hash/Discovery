It takes a few parameters and we’re going to quickly cover them.

-IP [string[]]

This is the only mandatory parameter, which is one more IP addresses, typically it will be an array of all usable IPs on your subnet. There are a few ways to generate this, one is to use my PSNetAddressing module, otherwise if you only need to scan a /24, the following is an easy option.

$IPs = 1..254 | % {"10.250.1.$_"}
-DelayMS [int]

This is a delay in milliseconds between packets being sent. I discuss it in more detail further below, but consider increasing it if you’re on an unreliable LAN or poor Wi-Fi connection where latency and packet drops may be occurring. I found a 1ms delay to provide reliable results, but the default is set to 2 as an extra buffer.

-ClearARPCache

This is simply a switch, it clears the ARP cache before starting the scan. It is recommended, but does require an administrative prompt.

Examples
The following examples assume an array called $IP was previously created with all usable IP addresses on the subnet.

PS C:\> Find-LANHosts -IP $IPs

IP           MACAddress
--           ----------

10.250.1.1   6c-41-6a-54-4d-0c
10.250.1.25  cc-f9-57-30-29-e2
10.250.1.101 00-a0-96-f4-c9-0d
10.250.1.102 58-f3-9c-63-3c-39
10.250.1.103 20-c9-d0-93-b5-04
10.250.1.104 6c-c7-ec-56-a1-57
10.250.1.105 08-c5-e1-ca-43-c8
10.250.1.112 00-15-65-ab-30-3b
PS C:\> Find-LANHosts $IPs -DelayMS 5 -ClearARPCache

IP           MACAddress
--           ----------
10.250.1.1   6c-41-6a-54-4d-0c
10.250.1.25  cc-f9-57-30-29-e2
10.250.1.101 00-a0-96-f4-c9-0d
10.250.1.102 58-f3-9c-63-3c-39
10.250.1.103 20-c9-d0-93-b5-04
10.250.1.104 6c-c7-ec-56-a1-57
10.250.1.105 08-c5-e1-ca-43-c8
10.250.1.112 00-15-65-ab-30-3b
Execution time
One of my favorite things about this is the speed.

PS C:\> Measure-Command {Find-LANHosts -IP $IPs} | Select -ExpandProperty TotalSeconds
0.7829996
That’s under a second to scan an entire subnet with 254 IPs!

If you’ve looked at the code, there may be a lot of “Why are you doing it like that…?”. For those interested in some of the details, let’s get into it.

Interpacket delays
Why do we need the DelayMS parameter? Firstly, we can set it to 0 which bypasses the delay mechanism (and runs even faster :)), meaning the function will send out packets as quickly as it possibly can, but I would only recommend doing this on extremely reliable networks with quality equipment. Here’s why.

Most of the testing I’ve done has been on my home network which frankly isn’t great. The only decent piece of equipment is a Cisco router, the rest consists of an old 8 port switch, a wireless AP from a decade ago, and worst of all, PowerLine (Ethernet over Power) adapters which link the office to where our router and AP are. To top it off, I’ve got a solar inverter that is mounted outside on the other side of a brick wall, and it’s on the Wi-Fi network.

All of this resulted in missing packets when I was running tests with no delay. Wireshark would show a stream of outgoing ARP requests, but one or two devices wouldn’t return anything. Something along the path, or the devices themselves were not coping with a blast of ARP broadcasts. I didn’t bother investigating too deeply as it didn’t matter what the cause was - if I was having the issue, others might too, so it needed to be addressed in the code.

Adding a 1 millisecond delay between packets solved the issue for me, and the default is set to 2 ms to provide an extra buffer.

Lastly, while this script is blazingly fast, increasing the DelayMS value too much can cause the script to run for longer than 15 seconds (i.e., Number of Hosts * Delay can exceed 15 seconds), at which point Windows may begin discarding old ARP records, meaning we won’t see them in the output. In the unlikely event that the script runs for more than 15 seconds, a warning will be displayed.

We’re using ARP to detect hosts, so why is there UDP?
Much of it comes down to performance and code cleanliness. .NET has a SendARP function within the tools provided by the IP Helper Win32 API, but it has a couple of shortcomings.

The first issue is that there is no way to set a timeout value, or the number of ARP requests sent. ARP generally times out quickly, so this wasn’t a show stopper, until you get to the next problem.

The bigger issue is that the function is synchronous, that is, it processed one IP at a time, and would only move onto the next when a reply was received (best case scenario), or when all 3 packets had been sent and had timed out (worst case scenario). If we wanted parallelisation we would have to implement it ourselves.

For PowerShell, this generally means Jobs or Runspaces (which I’ve blogged about before). I initially wrote this function using SendARP and Runspace Pools, and got the execution time down to 4.5 seconds for a /24 scan. This was acceptable, but using runspaces felt too heavy, and it made the code unnecessarily complicated.

For a while I figured that was the best we could do, but then Tobias Weltner wrote a fantastic article on using Win32_PingStatus to asynchronously ping scan a subnet. I ran this on a /24 and it completed in 1.5 seconds.

What?

The best I could do, with only ARP, not having to wait for any ICMP replies, was 4.5 seconds, and here was a guy with a function that did it in 1.5 with the extra overhead. I was impressed, and it actually put a big smile on my face, but it also stung a bit 😅.

I took this as a challenge and started looking for other .NET methods that I could use, preferably something asynchronous that didn’t involve needing to wait for replies. ICMP Echos were out, TCP would result in a 3 way handshake if whatever port we connected to happened to be open (and I wanted to avoid an avalanche of SYN packets to begin with), so it came down to UDP.

I did first look at whether it was possible craft raw packets, but this seemed to involve third party libraries and a whole lot of unnecessary complexity.

I finally came to discover System.Net.Sockets.Udpclient and that fit the bill perfectly - it is asynchronous and incredibly quick to implement. Remember, we don’t care about the UDP packets at all, what we’re after is the underlying functionality that generates ARP requests for unknown host MAC addresses. The UDP packets on top of that are of no consequence.

Why System.Threading instead of Start-Sleep
We can expect a bit of overhead when using PowerShell cmdlets vs .NET methods, but what I witnessed with Start-Sleep blew me away.

How long would you expect the following code to take?

1..100 | % {Start-Sleep -Milliseconds 1}
A bit over 100 ms, right? Let’s take a look.

PS C:\> Measure-Command {1..100 | % {Start-Sleep -Milliseconds 1}} | Select -ExpandProperty TotalMilliseconds
1558.6298
One and a half seconds! Let’s try exactly the same thing with System.Threading

Measure-Command {1..100 | % {[System.Threading.Thread]::Sleep(1)}} | Select -ExpandProperty TotalMilliseconds
202.3078
There appears to be around 14ms of overhead for every Start-Sleep call.

PS C:\> 1..100 | % { (Measure-Command {Start-Sleep -Milliseconds 1} | Select -ExpandProperty TotalMilliseconds) - 1} | Measure-Object -Average

Count    : 100
Average  : 14.209163
What we’re doing here is measuring how long every Start-Sleep -Milliseconds 1 takes, subtracting 1 from the result (to account for our sleep), and averaging all output values. This isn’t by any means scientific, and the results may vary somewhat between computers, but System.Threading provides much better performance.

PS C:\> 1..100 | % {(Measure-Command {[System.Threading.Thread]::Sleep(1)} | Select -ExpandProperty TotalMilliseconds) - 1} | Measure-Object -Average

Count    : 100
Average  : 0.751701
Why arp -a instead of Get-NetNeighbor
PowerShell has a great cmdlet which outputs the ARP cache as an object, Get-NetNeighbor, and also provides a lot more detail on each cache entry, but arp -a proved to be faster, even with string manipulation and the parsing through ConvertFrom-CSV.

That’s all for now, hope you enjoyed the read if you got this far!
