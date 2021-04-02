# AWS-IPv4-IPv6-Dual-Stack-EC2-Instances
AWS IPv4/IPv6 Dual Stack EC2 Instances
<p>When it comes to IPv6 support, AWS has been
surprisingly slow in its adoption.  EC2 only gained
support for IPv6 <a
href="https://aws.amazon.com/about-aws/whats-new/2017/01/announcing-internet-protocol-version-6-ipv6-support-for-ec2-instances-in-amazon-virtual-private-cloud-vpc-regional-expansion/">in
2017</a> (!), and Elastic IPs are <em>still</em> <a
href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html#ip-addressing-eips">IPv4
only</a>.  Even basic dualstack support for
EC2 instances requires the creation of a <a
href="https://aws.amazon.com/vpc/">VPC</a> and a number
of configuration steps</a> that are not trivial for
novice users to implement.</p>

<p>If you've had your AWS account for long enough
(i.e., you're on "<a
href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-classic-platform.html">EC2-Classic</a>"),
you don't even have a default VPC and thus can't
enable IPv6 on that.  For that reason, I've been
going back to using an
IPv6 tunnelbroker</a> to enable IPv6 on my default EC2
instances when I wanted to create a dualstack
environment.</p>

<p>Anyway, since I'll forget how to do this or where
to find the right commands, here are my notes on
creating a dualstack VPC in the hopes that somebody
else may find them useful.  At the bottom, you'll also
find a single script to run all commands in one go:</p>

<hr width="75%">

<h2><a name="new-vpc">Create a new VPC</a></h2>

<p>In the UI, go to 'Services'-&gt;'Network &amp;
Content Delivery' -&gt; 'VPC' -&gt; 'Your VPCs'.
Select 'Create VPC'.</p>

<p>Let's call it "dualstack", give it an RFC1918 CIDR of
your liking (let's say "10.10.0.0/24"), select "Amazon
provided IPv6 CIDR block", and "Default" tenancy.</p>

<p><a
href="https://console.aws.amazon.com/vpc/home"><img
src="images/ec2ipv6/create-vpc.png" alt="AWS Console:
Create VPC" border="0"></a></p>

<p>After creation of the VPC, select it and 'Enable
DNS hostnames' and 'Enable DNS resolution'.  (Note:
DNS hostnames are only added with IPv4 addresses.  As
of May 2020, Amazon <a
href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html">says</a>:
<em>We do not support IPv6 DNS hostnames for your
instance.</em>)</p>

<p>Via the command-line, we'll use JSON output as the
default (<tt>output = json</tt> in
<tt>~/.aws/config</tt>) and then utilize the versatile
command.  To create the VPC, you'd
then run:</p>

<div style="text-align: center;"><pre class="code">$ export VPCID="$(aws ec2 create-vpc --cidr-block 10.10.0.0/24 --amazon-provided-ipv6-cidr-block | jq -r '.Vpc.VpcId')"
$ aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${VPCID}"
$ aws ec2 modify-vpc-attribute --enable-dns-hostnames --vpc-id "${VPCID}"
$ aws ec2 modify-vpc-attribute --enable-dns-support --vpc-id "${VPCID}"</pre></div>

<h2><a name="new-gateway">Create a new Internet Gateway</a></h2>

<p>Hosts on our VPC want to talk to the internet, so
let's create an internet gateway.  To create a new
Internet Gateway using the UI, go to
'Services'-&gt;'Network &amp; Content Delivery' -&gt;
'VPC' -&gt; 'Internet Gateways'. Select 'Create
Internet Gateway'.  For consistency, let's name it
"dualstack" as well.</p>

<p><a
href="https://console.aws.amazon.com/vpc/home#Create%20Internet%20Gateway:"><img
src="images/ec2ipv6/create-igw.png" alt="AWS Console:
Create Internet Gateway" border="0"></a></p>

<p>Via the command-line, that'd be:</p>

<div style="text-align: center;"><pre class="code">$ export IGW="$(aws ec2 create-internet-gateway | jq -r '.InternetGateway.InternetGatewayId')"
$ aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${IGW}"</pre></div>

<h2><a name="attach-gateway">Attach the Gateway to your VPC</a></h2>

<p>In the UI, select the newly created Internet
Gateway, then select 'Actions'-&gt;'Attach to VPC' and
select the newly created VPC.</p>

<p><a
href="https://console.aws.amazon.com/vpc/home#Attach%20to%20VPC"><img
src="images/ec2ipv6/attach-gw.png" alt="AWS Console:
Attach Internet Gateway to VPC" border="0"></a></p>

<p>Via the command-line, that'd be:</p>

<div style="text-align: center;"><pre class="code">$ aws ec2 attach-internet-gateway --vpc-id "${VPCID}" --internet-gateway-id "${IGW}"</pre></div>

<h2><a name="create-subnet">Create a new Subnet</a></h2>

<p>Our hosts need to live somewhere in our VPC, so
let's create a new subnet. This will be a dualstack
subnet with both IPv4 and IPv6 addresses and
connectivity to the internet.</p>

<p>In the UI, go to 'Services'-&gt;'Network &amp;
Content Delivery' -&gt; 'VPC' -&gt; 'Subnets'.  Select
'Create subnet', give it the "dualstack" name tag, and
pick your newly created VPC "dualstack". Next, carve
out a suitable CIDR from your /24 for this subnet;
let's say, a /26 (i.e., "10.10.0.0/26"). Select
'Custom IPv6' and fill in '00' for the IPv6 /64
CIDR.</p>

<p><a
href="https://console.aws.amazon.com/vpc/home#CreateSubnet:"><img
src="images/ec2ipv6/create-subnet.png" alt="AWS Console:
Create Subnet" border="0"></a></p>

<p>Next, select the newly created subnet and choose
"Modify auto-assign IP settings" to enable a public
IPv4 and a public IPv6 address:</p>

<p><a
href="https://console.aws.amazon.com/vpc/home#ModifyAutoAssignIpSettings:"><img
src="images/ec2ipv6/modify-subnet.png" alt="AWS Console:
Modify Subnet" border="0"></a></p>

<p>Via the command-line, we first need to determine
the IPv6 CIDR associated with your VPC.  AWS assigns a
/56 by default, so we'll carve that up into a
reasonably sized /64:</p>

<div style="text-align: center;"><pre class="code">$ export V6CIDR=$(aws ec2 describe-vpcs --vpc-id "${VPCID}" | \
         jq -r '.Vpcs[].Ipv6CidrBlockAssociationSet[].Ipv6CidrBlock')
$ export SUBNET="$(aws ec2 create-subnet --cidr-block 10.10.0.0/26 \
        --ipv6-cidr-block ${V6CIDR%/56}/64 \
        --vpc-id "${VPCID}" | jq -r '.Subnet.SubnetId')"
$ aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${SUBNET}"
$ aws ec2 modify-subnet-attribute --subnet-id "${SUBNET}" --assign-ipv6-address-on-creation
$ aws ec2 modify-subnet-attribute --subnet-id "${SUBNET}" --map-public-ip-on-launch</pre></div>

<h2><a name="create-rtb">Add a new Route Table</a></h2>

<p>Subnets are all nice and well, but in order to be
able to talk to anything beyond the local network, you
need some sort of routing. So let's create a new Route
Table.</p>

<p>In the UI, go to 'Services'-&gt;'Network &amp; Content
Delivery' -&gt; 'VPC' -&gt; 'Route Tables'. Select 'Create
route table', name it "dualstack", and attach it to
the "dualstack" VPC you created above.</p>

<p><a
href="https://console.aws.amazon.com/vpc/home#CreateRouteTable:"><img
src="images/ec2ipv6/create-rtb.png" alt="AWS Console:
Create Route Table" border="0"></a></p>

<p>Via the command-line, that'd be:</p>

<div style="text-align: center;"><pre class="code">$ export RTB="$(aws ec2 create-route-table --vpc-id "${VPCID}" | \
        jq -r '.RouteTable.RouteTableId')"
$ aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${RTB}"</pre></div>

<h2><a name="associate-rtb">Associate your Route Table with your Subnet</a></h2>

<p>The Route Table needs to be associated with the
subnet you created above. In the UI, select the newly
created Route Table, then select 'Actions' -&gt; 'Edit
Subnet Associations', select your "dualstack" subnet
and save.</p>

<p><a
href="https://console.aws.amazon.com/vpc/home#EditRouteTableSubnetAssociations:"><img
src="images/ec2ipv6/associate-rtb.png" alt="AWS Console:
Edit Route Table Subnet Associations" border="0"></a></p>

<p>Via the command-line, that'd be:</p>

<div style="text-align: center;"><pre class="code">$ aws ec2 associate-route-table --route-table-id "${RTB}" \
        --subnet-id "${SUBNET}" | jq '.AssociationId'</pre></div>

<h2><a name="create-routes">Create Route Table Routes</a></h2>

<p>A Route Table is all nice and well, but it's not
very useful without any routes. Let's add some!</p>

<p>In the UI, select the route table you just created,
then select 'Actions'-&gt;'Edit routes', then 'Add route'
and enter "0.0.0.0/0" and select the Internet Gateway
you created above as the target. Then repeat the same
for IPv6 by using "::/0" as the destination with the
same Internet Gateway.</p>

<p><a
href="https://console.aws.amazon.com/vpc/home#EditRoutes:"><img
src="images/ec2ipv6/edit-routes.png" alt="AWS Console:
Edit Routes" border="0"></a></p>

<p>Via the command-line, that'd be:</p>

<div style="text-align: center;"><pre class="code">$ aws ec2 create-route --route-table-id "${RTB}" --destination-ipv6-cidr-block ::/0 --gateway-id "${IGW}"
$ aws ec2 create-route --route-table-id "${RTB}" --destination-cidr-block 0.0.0.0/0 --gateway-id "${IGW}"</pre></div>

<h2><a name="create-sg">Create a security group</a></h2>

<p>We need to create a new security group for our EC2
instances. Since security groups are stateful, we
don't need to create rules for established TCP
connections as you would have to for network ACLs and
can only worry about which ports we want to allow
traffic into.</p>

<p>In this example, since we are looking to
demonstrate what a full dualstack host looks like on
the internet, we want to allow <em>any and
all</em> traffic.  For your purposes, you may wish to
restrict the traffic you allow to come in, but you'll
have to remember to add rules for both IPv4 and
IPv6!</p>

<p>In the UI, go to 'Services' -&gt; 'Network &amp; Content
Delivery' -&gt; 'VPC' -&gt; 'Security Groups'. Select
'Create security group', name it "dualstack", give it
a description, and associate it with your newly
created VPC. After that, select 'Actions'-&gt;'Edit
inbound rules'-&gt;'Add rule' and permit TCP
traffic:</p>

<p><a
href="https://console.aws.amazon.com/vpc/home#CreateSecurityGroup:"><img
src="images/ec2ipv6/create-sg.png" alt="AWS Console:
Create Security Group" border="0"></a></p>

<p><a
href="https://console.aws.amazon.com/vpc/home#CreateSecurityGroup:"><img
src="images/ec2ipv6/sg-rules.png" alt="AWS Console:
Edit Security Group Rules" border="0"></a></p>

<p>Via the command-line, that'd be:</p>

<div style="text-align: center;"><pre class="code">$ export SG="$(aws ec2 create-security-group --description "Default security group for dualstack instances" \
        --group-name "dualstack" \
        --vpc-id "${VPCID}" | jq -r '.GroupId')"
$ aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${SG}"
$ aws ec2 authorize-security-group-ingress --group-id "${SG}" \
        --ip-permissions '[{"IpProtocol": "-1", "IpRanges" : [{"CidrIp": "0.0.0.0/0"}], "Ipv6Ranges" : [{"CidrIpv6": "::/0"}]}]'</pre></div>

<h2><a name="launch">Launch an instance</a></h2>

<p>You're now ready to launch a dualstack instance.
To do so, you need to remember to specify the correct
security group as well as subnet.  For example, to
launch a <a
href="https://www.freebsd.org/releases/12.1R/announce.html">FreeBSD
12.1</a> instance:</p>

<div style="text-align: center;"><pre class="code">$ export ID=$(aws ec2 run-instances --image-id ami-0de268ac2498ba33d \
        --instance-type t2.micro --count 1 --security-group-ids "${SG}" \
        --subnet-id "${SUBNET}" | \
        jq -r '.Instances[].InstanceId')
$ aws ec2 describe-instances --instance-id "${ID}" | \
        jq -r '.Reservations[].Instances[] |
                "\(.PublicDnsName) \(.PublicIpAddress) \(.NetworkInterfaces[].Ipv6Addresses[].Ipv6Address)"'
ec2-54-210-62-255.compute-1.amazonaws.com 54.210.62.255 2600:1f18:400c:b800:380f:b42a:505d:4fab</pre></div>

<p>Wait for it to come up, and low and behold:</p>

<div style="text-align: center;"><pre class="code">$ ping6 -c 3 2600:1f18:400c:b800:380f:b42a:505d:4fab
PING6(56=40+8+8 bytes) 2001:470:30:84:e276:63ff:fe72:3900 --&gt; 2600:1f18:400c:b800:380f:b42a:505d:4fab
16 bytes from 2600:1f18:400c:b800:380f:b42a:505d:4fab, icmp_seq=0 hlim=45 time=7.943 ms
16 bytes from 2600:1f18:400c:b800:380f:b42a:505d:4fab, icmp_seq=1 hlim=45 time=12.318 ms
16 bytes from 2600:1f18:400c:b800:380f:b42a:505d:4fab, icmp_seq=2 hlim=45 time=8.776 ms

--- 2600:1f18:400c:b800:380f:b42a:505d:4fab ping6 statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/std-dev = 7.943/9.679/12.318/2.323 ms

$ ssh ec2-user@ec2-54-210-62-255.compute-1.amazonaws.com "ifconfig xn0"
xn0: flags=8843&lt;UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST&gt; metric 0 mtu 9001
        options=503&lt;RXCSUM,TXCSUM,TSO4,LRO&gt;
        ether 06:48:62:d7:64:6f
        inet6 fe80::448:62ff:fed7:646f%xn0 prefixlen 64 scopeid 0x2
        inet6 2600:1f18:400c:b800:380f:b42a:505d:4fab prefixlen 128
        inet 10.10.0.38 netmask 0xffffffc0 broadcast 10.10.0.63
        media: Ethernet manual
        status: active
        nd6 options=23&lt;PERFORMNUD,ACCEPT_RTADV,AUTO_LINKLOCAL&gt;
$ </pre></div>

<h2><a name="multi-region">Supporting multiple regions</a></h2>

<p>So far, so good.  But this is still a pain, because
while with the (IPv4-only) defaults, you can trivially
launch an instance in any region so long as you know
the AMI ID, but if you want to launch a dualstack
instance in a given region, you do of course have to
repeat the above setup for that region, and then
specify the correct subnet and security group
again.</p>

<p>For example, suppose your default region is
<tt>us-east-1</tt>, but you wish to launch an instance
in <tt>sa-east-1</tt>, you'd first have to run through
all the steps above to create the "dualstack" VPC in
that region.  Fortunately, this is a one-time
thing.</p>

<p>But we're lazy, so let's create a <a
href="create-dualstack-vpc">simple script</a> so we
can easily run this for any region we wish to
support:</p>

<div style="text-align: center;"><pre class="code">$ cat &gt; ~/bin/create-dualstack-vpc &lt;&lt;"EOF"
#! /bin/sh

set -eu

export VPCID="$(aws ec2 create-vpc --cidr-block 10.10.0.0/24 --amazon-provided-ipv6-cidr-block | jq -r '.Vpc.VpcId')"
aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${VPCID}"
aws ec2 modify-vpc-attribute --enable-dns-hostnames --vpc-id "${VPCID}"
aws ec2 modify-vpc-attribute --enable-dns-support --vpc-id "${VPCID}"

export IGW="$(aws ec2 create-internet-gateway | jq -r '.InternetGateway.InternetGatewayId')"
aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${IGW}"
aws ec2 attach-internet-gateway --vpc-id "${VPCID}" \
        --internet-gateway-id "${IGW}"

export V6CIDR=$(aws ec2 describe-vpcs --vpc-id "${VPCID}" | \
        jq -r '.Vpcs[].Ipv6CidrBlockAssociationSet[].Ipv6CidrBlock')
export SUBNET="$(aws ec2 create-subnet --cidr-block 10.10.0.0/26 \
        --ipv6-cidr-block ${V6CIDR%/56}/64 \
        --vpc-id "${VPCID}" | jq -r '.Subnet.SubnetId')"
aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${SUBNET}"
aws ec2 modify-subnet-attribute --subnet-id "${SUBNET}" --assign-ipv6-address-on-creation
aws ec2 modify-subnet-attribute --subnet-id "${SUBNET}" --map-public-ip-on-launch

export RTB="$(aws ec2 create-route-table --vpc-id "${VPCID}" | \
        jq -r '.RouteTable.RouteTableId')"
aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${RTB}"
aws ec2 associate-route-table --route-table-id "${RTB}" \
        --subnet-id "${SUBNET}" &gt;/dev/null
aws ec2 create-route --route-table-id "${RTB}" --destination-ipv6-cidr-block ::/0 --gateway-id "${IGW}" &gt;/dev/null
aws ec2 create-route --route-table-id "${RTB}" --destination-cidr-block 0.0.0.0/0 --gateway-id "${IGW}" &gt;/dev/null

export SG="$(aws ec2 create-security-group --description "Default security group for dualstack instances" \
        --group-name "dualstack" \
        --vpc-id "${VPCID}" | jq -r '.GroupId')"
aws ec2 create-tags --tags Key=Name,Value=dualstack --resources "${SG}"
aws ec2 authorize-security-group-ingress --group-id "${SG}" \
        --ip-permissions '[{"IpProtocol": "-1", "IpRanges" : [{"CidrIp": "0.0.0.0/0"}], "Ipv6Ranges" : [{"CidrIpv6": "::/0"}]}]'
EOF
$ chmod a+rx ~/bin/create-dualstack-vpc
</pre></div>

<p>Now the next annoying thing is that you now have
different subnet- and security-group IDs in each
region, so if you want to spin up a dualstack
instance, you need to remember which subnet goes with
which region.  What a pain.  Good thing we labeled our
resources consistently, so that we can now on-demand
search for "dualstack" subnets or security groups and
create a shell function to grab the right id from that
label.  For example:</p>

<div style="text-align: center;"><pre class="code">$ cat &gt;&gt; ~/.bashrc &lt;&lt;"EOF"

startInstance() {
        subnet=$(aws ec2 describe-subnets | jq -r '.Subnets[] | select( .Tags[]? | select(.Value == "dualstack")).SubnetId')
        sg=$(aws ec2 describe-security-groups | jq -r '.SecurityGroups [] | select( .GroupName == "dualstack").GroupId')
        aws ec2 run-instances --security-group-ids "${sg}" \
                 --subnet-id "${subnet}" \
                 --image-id $@
}
 
alias start-freebsd="startInstance ami-0de268ac2498ba33d --instance-type t2.micro"
alias start-freebsd-sa="startInstance ami-0c01daaa164ea42de --instance-type t3.micro"
EOF
$ </pre></div>

<p>With all those bits in place, we can now create
dualstack FreeBSD instances in our default region as
well as in <tt>sa-east-1</tt>:</p>

<div style="text-align: center;"><pre class="code">$ aws configure set region sa-east-1
$ ~/bin/create-dualstack-vpc
$ start-freebsd-sa | jq -r '.Instances[].InstanceId'
i-0d88d896a110bdc7b
$ aws configure set region us-east-1
$ start-freebsd | jq -r '.Instances[].InstanceId'
i-0657d197cf9a1ac4a
$ </pre></div>

<p>
And there you have it.  Spinning up a dualstack EC2
instance is now somewhat easier.  Perhaps in another
15 years IPv6 will finally be a first-class citizen on
the internet, including AWS...
</p>
