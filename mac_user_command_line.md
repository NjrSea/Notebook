# mac下用户 用户组命令行操作

[linke](http://www.cnblogs.com/zhuiluoyu/p/5455919.html)

使用mac的时候需要像linux一样对用户和群组进行操作，但是linux使用的gpasswd和usermod在mac上都不可以使用，mac使用dscl来对group和user操作。

查看用户组：

dscl . list /groups
 查看用户：

dscl . list /users
 添加用户组：

sudo dscl . -create /Groups/test
 删除用户组：

sudo dscl . -delete /Groups/test
 添加用户：

sudo dscl .  -create /Users/redis
 删除用户：

sudo dscl . -delete /Users/redis
 显示所有users对应的group:

sudo dscl . -list /groups GroupMembership
 添加user到group:

sudo dscl . -append /Groups/groupname GroupMembership username
从group中删除user：

sudo dscl . -delete /Groups/groupname GroupMembership username
 other:
dscl . -create /Groups/GROUP
dscl . -create /Groups/GROUP PrimaryGroupID GID
dscl . -create /Groups/GROUP Password \*
dscl . -create /Users/USER
dscl . -create /Users/USER UniqueID UID
dscl . -create /Users/USER UserShell /usr/bin/false
dscl . -create /Users/USER RealName 'DESCRIPTION'
dscl . -create /Users/USER NFSHomeDirectory DIRECTORY
dscl . -create /Users/USER PrimaryGroupID GID
dscl . -create /Users/USER Password \*
 显示所有用户组的ID

1
dscl . -list /Groups PrimaryGroupID
 读取用户组的信息：

dscl . read /groups/wheel
 
结果：
AppleMetaNodeLocation: /Local/Default
GeneratedUID: ABCDEFAB-CDEF-ABCD-EFAB-CDEF00000000
GroupMembers: FFFFEEEE-DDDD-CCCC-BBBB-AAAA00000000
GroupMembership: root
Password: *
PrimaryGroupID: 0
RealName:
 System Group
RecordName: wheel
RecordType: dsRecTypeStandard:Groups
 读取用户组下的成员：

dscl . read /groups/wheel GroupMembership
 
结果：
GroupMembership: root
 读取用户信息：

dscl . read /users/root
 
结果：
 
AppleMetaNodeLocation: /Local/Default
GeneratedUID: FFFFEEEE-DDDD-CCCC-BBBB-AAAA00000000
NFSHomeDirectory: /var/root
Password: *
PrimaryGroupID: 0
RealName:
 System Administrator
RecordName:
 root
 BUILTIN\Local System
RecordType: dsRecTypeStandard:Users
SMBSID: S-1-5-18
UniqueID: 0
UserShell: /bin/sh
 
dscl . read /users/root NFSHomeDirectory
 
结果：
NFSHomeDirectory: /var/root
