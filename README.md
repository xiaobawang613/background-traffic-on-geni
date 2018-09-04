# Generating realistic background traffic on GENI


## Background

Network researchers often test new developments with experiments where by they send a specific stream of traffic over an experimental network and measure the experimental traffic's performance. However, their results can be skewed by leaving out background traffic, which would run alongside the experimental traffic in a realistic network setting, or by running  unrealistic background traffic patterns. Thus, to accurately test the experiment results they are trying to find, researchers need to not only include background traffic in their experiments, but also to generate traffic that is as similar as possible to real background traffic.

## Results

After looking through various papers that used background traffic, we have noted two major trends in the traffic that they use. For one, we noticed that several papers sent a stream of overly simplified, synthetically generated traffic using iPerf, a tool used for performance measuring and tuning. IPerf traffic generally resembles the following, where the number of bytes/second is generally steady with occassional bursts. 
--graph of iPerf traffic

Alternatively, researchers who want to send more burstier traffic rely on a probabilist model of traffic. Shown below is a Wireshark I/O graph of traffic generated by D-ITG, one of the most often cited traffic generators. 
--graph of D-ITG traffic

Despite being more complicated, probabilistic traffic still differs from real application traffic. For example, in the following trace of all the traffic in a network, traffic is not only more complicated, but it is also split between "purple" traffic – traffic from the internet to the user– and "blue" traffic – traffic from the user to the internet.
--graph of composite traffic

If we compare specific application traces, we can see that traffic varies depending on the application. For example, traffic sent on Netflix is different than traffic sent on Skype. In the Netflix trace, more traffic from the internet to the user was sent out than traffic from the user to the internet. However, in the Skype trace, we see more traffic from the user to the internet. 
--graph of netflix and skype traffic

Thus, while researchers most often cite synthetically generated traffic and probablistic traffic, these traffic patterns are unrealistic and may still alter experiment results. In one of the papers we looked at, researchers ran three different types of background traffic: a constant stream, a modified but still constant stream, and a real traffic trace from the 2016 anonymized CAIDA dataset. They noticed that “the [CAIDA] traffic traces were burstier than the [constant bit rate] traffic” and “bursty cross traffic resulted in noisier measurement data than [constant bit rate traffic]”.

While this paper did play a real traffic trace, others understandably did not. After all, it can be time-consuming and difficult to find a promising traffic dataset and to learn to use the multi-step tools needed to replay the traffic. 

Our solution is to choose the most reliable datasets we could find and put a "wrap" around all of the steps involed in traffic replay tools. To make our tool available to everyone, we made the source code, the tool, and the instructions publically available on our GitHub blog. We hope that, by creating this tool, researchers can have an easy-to-use and realistic background traffic replay tool to produce more accurate experimental results. 

> Han Wang, Ki Suh Lee, Erluo Li, Chiun Lin Lim, Ao Tang, and Hakim Weatherspoon. 2014. Timing is Everything: Accurate, Minimum Overhead, Available Bandwidth Estimation in High-speed Wired Networks. In Proceedings of the 2014 Conference on Internet Measurement Conference (IMC ’14). 407–420

## Run my experiment


### Set up topology

In the GENI Portal, create a new slice and press "Add Resources." Scroll down to the "Choose RSpec" section, select the "URL" option, and load the RSpec from the following URL: [https://raw.githubusercontent.com/Zoe2140wu/background-traffic-on-geni/master/replaywrapperRspec.xml](https://raw.githubusercontent.com/Zoe2140wu/background-traffic-on-geni/master/replaywrapperRspec.xml)

This will load the following topology into your canvas:

<img src = "https://user-images.githubusercontent.com/23636883/44962385-60d73980-aeed-11e8-931d-de8dd93b63dc.png">

In this topology, researchers run experimental traffic through the local and internet nodes and background traffic from the tcpreplay node. For example, traffic being run from the local node to the internet node will first stop at the router and queue up with client-server background traffic sent from the tcpreplay node. After mixing as it would in a real network, the experimental traffic is directed to the internet node and background traffic is directed to a "dummy" MAC addresses. We will later set up these destination IP addresses.

Click on the "Site 1" button and bind this RSpec to an InstaGENI site. Note that the node used for replaying traffic should be a raw PC IG, of which limited quantities are available - if you aren't able to satisfy this request at the first aggregate you try, you may need to try others. 

Press "Reserve Resources" at the very bottom of the page and wait for your resources to be ready to log in. 

When your resources are ready, SSH into your `tcpreplay` node. We have reserved extra disk space on this node in order to have enough room to save large traffic capture files, but you'll need to set it up.

Run 

```
sudo /usr/testbed/bin/mkextrafs /mnt
sudo chmod a+w /mnt
```

to mount your extra disk space. Your extra disk space should be available in the `/mnt` directory.


We will also have to install some software on the `tcpreplay` node. Run

```
sudo apt-get update
sudo apt-get -y install tcpreplay tshark
```

(you can choose "yes" when prompted during the setup).

Next, get the `replaywrapper` script from GitHub. In your `tcpreplay` node, run

```
sudo wget -P /usr/local/bin https://raw.githubusercontent.com/Zoe2140wu/background-traffic-on-geni/master/replaywrapper
sudo chmod a+x /usr/local/bin/replaywrapper
```

If you run `which replaywrapper`, you should see that the `replaywrapper` is ready to run from `/usr/local/bin/replaywrapper`.

We also need to set up the router so that it will redirect the background traffic between the "local network" and the "Internet". Later in the experiment, when you replay the background traffic, you will select the IP address that should appear as the source and destination IP address in the packet headers. Since in our topology, the "Internet" uses the address range 10.0.1.0/24, the "Internet" endpoint IP address for the background traffic should be in this range, and similarly, the "local" endpoint IP address should be in 10.0.2.0/24. However, the source and destination addresses for the background traffic should also be IP addresses that don't exist in the network. Since they don't exist, the router won't be able to resolve the IP addresses to MAC addresses using ARP, so we will add static ARP entries on the router for all of the endpoint IP addresses that we'll use for our background traffic.

The default endpoint IP addresses are 10.0.1.254 and 10.0.2.254, so if you're going to use the defaults, SSH into your router node and add static ARP entries with:

```
sudo arp -s 10.0.1.254 02:47:a9:bb:e0:d0
sudo arp -s 10.0.2.254 02:37:a9:bb:e0:d0
```

(these are fake MAC addresses!) If you're going to use other endpoint IP addresses, add static ARP entries on the router node for these as well.

### Acquire a dataset

In this experiment, we show how to use a dataset of background traffic [described here](http://www.unb.ca/cic/datasets/vpn.html) that was generated at the University of New Brunswick for the paper

> Gerard Drapper Gil, Arash Habibi Lashkari, Mohammad Mamun, Ali A. Ghorbani, "Characterization of Encrypted and VPN Traffic Using Time-Related Features", In Proceedings of the 2nd International Conference on Information Systems Security and Privacy(ICISSP 2016) , pages 407-414, Rome, Italy.

The dataset captures real traffic from end users as they use various applications for browsing, email, chat, video streaming, file transfers, and others.  The major advantages to using this dataset for background traffic are:

* You can specify what kind of applications should appear in your background traffic, and in what proportion.
* The traffic is recent, captured in 2016, and reflects modern Internet applications.

However, this dataset gives no indiciation of what proportion of each application *should* be in background traffic, so you will have to use good judgment in making those experiment decisions.

To acquire a copy of this dataset, email [cic@unb.ca](cic@unb.ca). You will receive in response a link to a download website and a username and password for accessing the download website.

Browse the download website to find the file containing the captures, which is in the "ISCX-VPN-NonVPN-2016" subdirectory and is titled "CompletePCAPs.zip". Right-click on the file link and copy the URL.

Then, on the `tcpreplay` node, move to the directory with extra space:

```
cd /mnt
```

and download this file using the command

```
wget --user USERNAME --password 'PASSWORD' URL
```
where in place of `USERNAME`, `PASSWORD`, and `URL`, you specify the username and password you received by email, and the URL for the "CompletePCAPs.zip" file that you just found in your browser.

Extract the capture files from the archive using

```
unzip CompletePCAPs.zip
```

and run `ls` to verify that you have extracted the files.

As an alternative, depending on the needs of your experiment, you may be more interested in having background traffic with a realistic mix of applications and not necessarily have fine-grained control over what kind of traffic is represented. In this case, you can use the dataset from the University of Twente, described in

> Barbosa, Rafael Ramos Regis, et al. "Simpleweb/University of Twente traffic traces data repository." Centre for Telematics and Information Technology University of Twente, Enschede, Technical Report (2010). PDF: [https://ris.utwente.nl/ws/files/5096467/traces.pdf](https://ris.utwente.nl/ws/files/5096467/traces.pdf)

Specifically, we will use the "Trace 6" data described as follows:

>  A 100 Mbit/s Ethernet link connecting an educational organization to the internet has been measured. This is a relatively small organization with around 35 employees and a ittle over 100 students working and studying at this site (the headquarter location of this organization). All workstations at this location ( 100 in total) have a 100Mbit/s Lan connection. The core network consists of a 1 Gbit/s connection. The recordings took place between the external optical fiber modem and the first firewall. The measured link was only mildly loaded during this period. These measurements are from May - June 2007.

The data is available at [https://traces.simpleweb.org/traces/TCP-IP/location6/](https://traces.simpleweb.org/traces/TCP-IP/location6/). To download this onto your `tcpreplay` node, move to the directory with extra space:

```
cd /mnt
```

and then run

```
wget https://traces.simpleweb.org/traces/TCP-IP/location6/loc6-20070501-2055.gz
wget https://traces.simpleweb.org/traces/TCP-IP/location6/loc6-20070523-0005.gz
wget https://traces.simpleweb.org/traces/TCP-IP/location6/loc6-20070531-2043.gz
wget https://traces.simpleweb.org/traces/TCP-IP/location6/loc6-20070615-1644.gz
```

To extract the traces from the archive files, run

```
gunzip loc6-20070501-2055.gz
gunzip loc6-20070523-0005.gz
gunzip loc6-20070531-2043.gz
gunzip loc6-20070615-1644.gz
```

Also add a file extension to each file:

```
mv loc6-20070501-2055 loc6-20070501-2055.pcap
mv loc6-20070523-0005 loc6-20070523-0005.pcap
mv loc6-20070531-2043 loc6-20070531-2043.pcap
mv loc6-20070615-1644 loc6-20070615-1644.pcap
```

### Play back background traffic

After acquiring the datasets, ssh into your tcpreplay node. To run the wrapper:
```
replaywrapper -<flag>=str
```
Where you replay <flag> with one of the following options:
  -f Input a pcap file
  -l Specify endpoint IP address for local host
  -i Specify endpoint IP address for internet host
  -d Run command in /mnt directory
  -e Pcap file extension name (either .pcap or .pcap.gz)

## Notes
Further along, we intend to imporoving the accuracy of our replayed background traffic more realistic and optimizing user input. For example, since tcpreplay is unable to send traffic at a high data rate, we want to experiment with a Raw PC IG computer or tune other parameters. Additionally, while we currently reply on tcpreplay's truncate option to send large packets, we hope to work more with tcpreplay's fragmentation option and include it in our code. We also want to experiment with tools such as Tmix since ots closed-loop, feedback-based system better reflects real traffic. Finally, we hope to explore other traffic datasets and topologies to offer more options for researchers.
