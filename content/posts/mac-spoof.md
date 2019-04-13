---
title: "More Better Wi-Fi, Please"
date: 2019-04-14T09:47:18+12:00
draft: false
toc: true
images:
  - "/images/ptitf.jpg"
tags: 
  - networking
  - wifi
  - mac
  - hotel
  - internet
---

![cover-img](https://images.unsplash.com/photo-1483389127117-b6a2102724ae?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1267&q=80)

So here’s a divisive one…

Have you ever experienced the first world misfortune of a rate limited public Wi-Fi that cuts you off after a) having exceeding upload/download limits, or b) overstaying your 30min welcome?

As a thought experiment, let's suggest that many of these networks rely upon a fairly lazy form of MAC-based authentication: a form authentication that tracks your MAC address through cyberspace and executes network rules against you by way of this ‘unique’ identifier. And furthermore, many laptop-based wireless cards enable you to modify this MAC address. How might one circumvent the interweb overlords that would otherwise deprive you of your viral Rickroll memes?

Brief interlude: For those unfamiliar with MAC addresses and unwilling to engage Google for enlightenment, here’s something I prepared earlier:

>“A media access control address of a device is a unique identifier assigned to a network interface controller. For communications within a network segment, it is used as a network address for most IEEE 802 network technologies, including Ethernet, Wi-Fi, and Bluetooth.” -- Wikipedia

In effect, this is saying the wireless module (Wi-Fi or Bluetooth card) in your device has a unique identifier bestowed upon it by its creator -- the manufacturer. And this identifier is used by many network protocols for doing useful things… Like blocking your memes.

Network Interface Controllers (NICs) hard-code their MAC address; however, in a convenient twist of fate, many drivers allow the MAC address perceived by the operating system to be modified quite easily. Enter ‘MAC Spoofing.’

# Level 1: Overstaying Our Welcome

Problem #1 comes about when a network severs our connection to the universe after having exceeded some sort of quota, be it time- or data-based. In this scenario, the orchestrator of the network has placed our address in an access control list that prevents further access until network rules are met: timeout or buyout.

This is an easy one to fix -- just get a new address!

What follows will be a series of Mac-based terminal commands (convenient, really, given the topic 😉), but I’ll endeavour to include Linux equivalents where I can. They shouldn’t diverge too much, anyway. If you’re team Windows, you’re on your own -- I’m sorry…

## Step 0: Dependencies

We’ll be using `ifconfig`, `openssl`, `sed`, `awk`, and `nmap` in the following. Most should be pre-installed builtins, but if you find yourself lacking any of the tools (likely `openssl` or `nmap`) install via homebrew (Mac only) with…

```
brew install {package_name}
```

## Step 1: Get & Set Your MAC Address

`ifconfig` is a useful built-in command on most Unix-based (Mac & Linux) operating systems that both gives us useful information about network interfaces, and allows us to configure them.

So, just run:

```
ifconfig {interface} ether
```

Where `{interface}` is your network card, usually `en0`, and `ether` requests the link-level (OSI model) address of the interface, aka the MAC address:

```
ifconfig en0 ether
```
  
Generates something akin to:

```
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
ether 39:42:b3:af:c3:99
```

This tells us that our MAC address is currently `39:42:b3:af:c3:99`. But that just got blocked by the feudal lords of the network, so let’s replace it.

This can be done simply enough with an extension of our prior command:

```
sudo ifconfig en0 ether <new_mac_address>  # sudo required to modify an interface
```

E.g.

```
sudo ifconfig en0 ether 49:47:4e:64:61:e9
```

This will bring the interface down briefly, re-program the link-level address, then bring it back up again. Voila! Your Wi-Fi access should automatically re-authenticate and give you fresh access to the network as a new user. You can double-check by re-running the first command. The result should match what you fed in.

## Step 2: Bonus Points

If you want to streamline this process a bit, you can have a new address automatically generated for you with the help of `openssl` for generating randomness, and `sed` for stream-editing into something useful. Give this a spin:

```
sudo ifconfig en0 ether $(openssl rand -hex 6 | sed 's/\(..\)/\1:/g; s/.$//')
```

# Level 2: Plus-Ones Welcome?

Ok, so we’re covered if we’re booted off a public network when the network decides we’ve overstayed. But what about hotels and other such establishments that ban you from the get-go unless you forfeit an extortion fee scaled in 10min time blocks. What then?!

Well, we have a solution for that, too.

But this scenario differs in that we must now identify as a user of the network that has already been authenticated. To achieve this, we’ll harness the powers of `nmap` to scan for active devices.

First, we need our IP address. Comparable to retrieving a MAC address with one minor tweak, run:

```
ifconfig en0 inet
```

To generate:

```
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
inet 192.168.1.60 netmask 0xffffff00 broadcast 192.168.1.255
```

This tells us that our current IP address is `192.168.1.60`. Use this to scan the network for other devices:

```
sudo nmap -sn {ip_address}/24  # sudo required to retrieve MAC addresses
```

E.g.

```
sudo nmap -sn 192.168.1.60/24
```

Telling `nmap` to complete a ping scan of our IP with something akin to a subnet mask of 24-bits so, in effect, “please scan IP addresses 192.168.1.0 through 192.168.1.255, thank-you.” This will give you a list of results (hopefully) that look something like this:

```
MAC Address: 10:a9:58:65:5a:c8 (Samsung Electronics)
Nmap scan report for 192.168.1.60
Host is up (0.079s latency).
MAC Address: dd:88:fa:5d:29:f8 (Google)
Nmap scan report for 192.168.1.61
Host is up (0.24s latency).
MAC Address: f0:c9:3b:9e:82:cf (Apple)
Nmap scan report for 192.168.1.62
Host is up (0.016s latency).
```

Now all that’s left is to adopt one of the given addresses -- easy peasy!

```
sudo ifconfig en0 ether f0:c9:3b:9e:82:cf  # Borrowing Apple’s address
```

Once your interface has dropped and re-authenticated, you should find your Twitch live-streaming, once again, unobscured.

# Level 3: Convenient Automation

In the spirit of convenience, this can be wrapped up into one tidy script available as a [gist](https://gist.github.com/jdh747/908110e353ff582071f533af2c909c4d).

It has 3 flag-induced usages:

1. `mac-spoof --set {mac_address}`: to set your MAC address as desired;
2. `mac-spoof --randomise`: to randomly assign a MAC address (Level 1);
3. `mac-spoof --select`: to scan for available MAC addresses and select from the list presented (Level 2);

{{< highlight bash >}}
#!/bin/sh

set -e

get_mac_address() {
	echo $(ifconfig en0 ether | grep ether | rev | cut -d' ' -f2 | rev)
}

get_ip_address() {
	echo $(ifconfig en0 inet | grep inet | cut -d' ' -f2)
}

set_mac_address() {
	sudo ifconfig en0 ether $1
}

set_branch() {
	echo "Updating mac address: $(get_mac_address) -> $1"
	set_mac_address $1
	echo "Done. New MAC: $(get_mac_address)"

}

randomise_branch() {
	echo "Current MAC: $(get_mac_address)"
	NEW_MAC=$(openssl rand -hex 6 | sed 's/\(..\)/\1:/g; s/.$//')
	set_mac_address ${NEW_MAC}
	echo "Done. New MAC: $(get_mac_address)"
}

select_branch() {
	IP_ADDRESS=$(get_ip_address)
	DEVICES=$(sudo nmap -sn ${IP_ADDRESS}/24 | awk '/Nmap/{ip=$NF;next} /MAC/{printf "%s|%s|", ip,$3} /MAC/{for(i=4;i<=NF;i++) printf "%s_", $i; printf ";"} ')
	
	# This part is needlessly clunky as, for whatever reason, the version of bash/sh on MAC misbehaves with arrays -- counts and index iteration didn't work properly
	i=0
	IFS=";" read -ra DEVICE_ARRAY <<< ${DEVICES}
	for DEV in ${DEVICE_ARRAY[@]}; do
		i=$((i+1)) && printf "\t%s) %s\n" "${i}" "${DEV}"
	done

	read -p "Select a device to spoof: " selected

	NEW_MAC=$(echo ${DEVICE_ARRAY} | cut -d' ' -f${selected} | cut -d \| -f2)
	echo "Spoofing MAC address: $(get_mac_address) -> ${NEW_MAC}"
	set_mac_address $NEW_MAC
	echo "Done. New MAC: $(get_mac_address)"
}

case $1 in
	--set)
		set_branch $2
		;;
	
	--randomise)
		randomise_branch
		;;

	--select)
		select_branch
		;;
	*)
		echo "Unknown or no option given"
		exit 64  # User error code
esac
{{< /highlight >}}

# Be Nice: A Word of Warning

I suggest being considerate with this knowledge and tailoring your use according to the types of paywalls you encounter. If, for instance, you’re confronted with a paywall that charges by the data consumed, it would be pretty darn inconsiderate to blitz a paying users quota.

Questions, qualms or something interesting to add? Start a conversation [here](https://twitter.com/joshhayes747/status/1117197531144830977?s=20)!

Be nice; have fun; de nada and good night!! 😀