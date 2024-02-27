POSIX ID mapping
================

POSIX systems are using numerical 32bit long integer value to assign IDs to
users and groups. Those IDs are used by the kernel to determine ownerships,
permissions and privileges.

To make life for human users more easy those IDs are mapped to user and group
names, traditionally in the file /etc/passwd and /etc/groups. In environments
with many users and computers using local files for the mapping does not scale
well and as a result centrally managed solutions where introduces. While
YP/NIS was specifically created for POSIX systems others like e.g. LDAP based
solutions can be used for other purposes as well. In RFC2307 and RFC2307bis it
is described how an LDAP server can be set up to allow POSIX systems to use
the LDAP data to map the IDs to names by specifying name and ID attributes and
even adds some other POSIX properties.

One of the most used LDAP implementation is Microsoft's Active Directory which
was introduced to centrally manage Windows user (and more). Given that the
obvious question is if it can be used to centrally manage POSIX users as well
and the answer is "yes, but ....".

There reasons for the "but" is that there are different approaches which all
have there individual pros and cons.

Active Directory
----------------
For example, since Active Directory is using LDAP, there is a solution based
on RFC2307/RFC2307bis which was added by Microsoft originally even with GUI
tools and migration tools from YP/NIS into Active Directory. But managing new
POSIX users and groups requires an extra effort from the AD administrators
because the POSIX attributes typically have to be added manually. Additionally
there is some ambiguity with respect to group-memberships because AD support
RFC2307 and RFC2307bis group-memberships in dedicated attributes. As a result
it would be possible to define three different sets of group memberships for a
user.

Samba and Winbind
+++++++++++++++++
But it is possible to make AD user and groups available to a POSIX system
without having POSIX attributes stored in AD. Samba was of course the first
with its winbind service to make AD users and groups available on a POSIX
system. One of the first solutions just assigned some POSIX UID and GID to AD
users and groups respectively and stored the relation in a local database.
Typically there was a starting number like e.g. 1000 and whenever a new POSIX
UID or GID was needed this number was incremented by one. This works fine on a
single system as long as the database is not lost. But since this is a
first-come-first-serve principal the same AD user or AD group might get
different POSIX IDs on different POSIX hosts even if they are joined into the
same AD domain.

To make the POSIX IDs more predictable other schemes were to the idmap-plugin
interface of winbind. One of them are RID-based schemes. The RID is the object
specific part of the SID and is a 32bit numerical value which typically starts
with 1000 and is incremented by one for every new user or group (or computer)
object. So it is quite similar to a POSIX ID but since there are typically
multiple domains in an AD forest and in each domain the RID counting will
start with 1000 this number cannot be used directly as POSIX ID. To get around
this individual, non-overlapping ID-ranges are defined for each domain and the
RID is used as an offset from the start of the ID-range so that the

    POSIX-ID = start_of_ID_range + RID

if the RID is smaller than the size of the ID-range. The details for each
ID-range have to be configured in Samba's smb.conf.

SSSD
++++
To allow lesser manual configuration SSSD use the same scheme but assigns the
ID-ranges for each domain automatically by splitting the available set of
POSIX-IDs into equally sized ranges with a default of 200k POSIX-IDs per
ID-range. The first ID-range starting with '0' is reserved for the local users
from /etc/passwd. Depending on the configurable range size there will be N
ID-ranges and SSSD will now take the domain-SID string of an AD domain, do a
hash/message digest calculation on this string and a modulo with N operation
on the result

     num = hash(domain_SID_string) % N

num is now an integer value between 0 and (N-1) and can be used to select
num'th ID-range created above for the domain with the given domain-SID. With
this this the POSIX-ID of an AD object from the domain with the given SID and
with a given RID would be:

    POSIX-ID = start_of_ID_range[num] + RID

So, basically the same, except that the ID-range is not configured but
selected automatically. The required parameters here are only the size of the
individual ID-ranges and the hash functions with a seed.

In case there are more objects in the AD domain than the given size of the
ID-ranges an additional ID-range is selected by adding '#1', '#2', '#3' etc to
the domain-SID string and doing the same calculation again.

There is a chance of a hash collision, i.e. the hash value of two different
domain-SID strings or (more probable) value after the modulo operation will be
the same and as a result the above scheme would assign the same ID-range to
two different domains. So far it looks that this does only happens very rarely
in real environment however one has to take it into account. To fix this a
(semi)-manual configuration can be used (this is planned for SSSD but not
implemented yet due to missing demand) or change a parameter of the
calculation like e.g. the range size, which is currently supported by SSSD.
The drawback of the second solution is that most probably the ID-ranges for
all domains will change and as a result existing POSIX-ID would change as
well.


Other types of Identity Providers
---------------------------------
The domain-SID and the RID are specific features of Active Directory which
might not be available with other sources of user and group information. A
classical example would be an LDAP server without RFC2307/RFC2307bis schema
available. A more modern example would be REST-based Identity Providers (IdPs)
which are used for OAuth2/OIDC authentication e.g. for web-application.

Here as well a range-based POSIX-ID mapping can be applied. The range itself
can either be manually configured or calculated as above but instead of a
domain-SID some other unique identifier should be used. An obvious candidate
would be a URI or URL, e.g. 'ldap://my.ldap.server' or 'https://my.idp.server'
as long as public resolvable DNS names are used in the URI/URL they are unique
by design. There might be issues with privately/internally used names collide
with public resolvable one but this is not recommended and might cause issues
at other places as well. So to select an ID-range we can reformulate

     num = hash(unique_identifier_of_ID_server) % N

Now we have to find an offset for a given object to calculate the POSIX-ID.
Unfortunately we cannot expect that an ID-service always provides a unique
integer identifier of suitable size like the RID in AD or a POSIX-ID
(typically there is a UUID or GUID which can be represented as an integer but
this will be a 128bit or longer integer, so not a suitable size, and since the
UUID/GUID does not contain a domain/service specific part like AD's SID they
cannot be used like SIDs as described above).

However, as long as there is such integer value, it would be possible to use
it (maybe with some manually configurable shift to avoid very large offsets in
case where this value will start with 1000000 or similar). But it has to be
noted that even if this number is added as POSIX-ID in the ID service it might
be better to not consider this value as absolute number and use it (as offset)
in the reserved ID-range starting with '0' but use it as offset for a
calculated or manually configured ID-range for the given ID-service to avoid
collisions with other ID-services which are either manually added or are made
available in federated environments.

Coming back to the more general case where the ID-service does not provide
such integer. Here we can use another attribute which is unique and available
for objects (at least for users and groups). Candidates might be UUID or GUID
as already mentioned. Another might be the canonical/login name which has to
be unique at least for an individual ID-service (or at least a tenant of it
with a unique URI/URL). To make this an offset we can do a hash/message digest
operation again and to a modulo operation with the range size:

    offset = hash(unique_attribute_value) % range_size

Hash collisions will be possible here as well, but there is currently no
experience how often this would happen. Nevertheless they have to be handled
somehow and the only way I can currently think of is by manually setting
offsets for the set of colliding objects.

Additionally, since hashes are not revertable, it will not be possible to find
a user or a group by UID or GID respectively if the ID wasn't calculated and
stored before.



To generalize the above:

Given an array of N non-overlapping ID-ranges defined by the start and the
size of each of the ranges a POSIX-ID can be calculated as

    
    POSIX-ID = start_of_ID_range[num] + offset

where

    num = hash(unique_identifier_of_ID_server) % N

and

    offset = f(unique_attribute_value) % size_of_ID_range[num]

where

    hash:
        (configurable) message digest function with, if needed,
        (configurable) seed

and

    f:
       configurable function, depending on the type of the unique attribute,
       can be the
           * identity-function for integer counter attributes or
           * a shifting function for very large integer counter attributes or
           * a message digest for generic attributes or
           * a special operation to extract a unique integer from some attribute, e.g. extract the RID from a SID string or
           * ...


Even if the range assignment is automatic here this can be combined with
manually configured ranges by making sure that ranges in the array of non-overlapping
ranges used for the calculation do not overlap with any of the manually
configured id-ranges.
