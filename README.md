# Vulnerability in apport daemon (a.k.a. whoopsie)
- An integer overflow in whoopsie 0.2.69, results in an out-of-bounds write to a heap allocated buffer when 'bytesNeeded' exceeds max of uint32.
- An exploit could allow the attacker to cause a denial of service (segmentation fault and crash).

## Basic
When a program has been crashed, Linux system tries to create a ‘.crash’ file on ‘/var/crash/’ directory with python script located in ‘/usr/share/apport/apport’. 
The file contains a series of system crash information including core dump, syslog, stack trace, memory map info, etc.
After the creation of ‘.crash’ file, apport daemon (a.k.a. whoopsie) extracts the above information from the ‘.crash’ file and encodes it into binary json (bson) format.
Lastly, the whoopsie forwards the data to a remotely connected Ubuntu Error Report system.
  
## Vulnerability
In .crash file, the contents are stored in a key-value format as follows:
```
ProblemType: Crash
Architecture: amd64
...
ProcMaps: 
 556ffb85a000-556ffb866000 r-xp 00000000 08:01 404793 …
...
```
whoopsie daemon separately measures the length of ‘key’ and ‘value’ in .crash file after finding .crash file. 
The summation of the length of ‘key’, ‘value’, and additional bytes is assigned to ‘bytesNeeded’ variable. 
The daemon attempts to allocate memory region according to the ‘bytesNeeded’.
An integer overflow can happen on a ‘bytesNeeded’ when the following requirements are met.
  - length of ‘value’ in .crash file => 0 < {length of ‘value’} < 1024
  - length of ‘key’ in .crash file => UINT32_MAX - {length of ‘value’} - 7 < {length of ‘key’} < UINT32_MAX

If so, unexpected small memory region is allocated and memory corruption can be happened.

## Attack Scenario
1) Create a fake.crash file
- ‘.crash’ file is composed of the following format: ‘key : value’.
- As I mentioned before, the followings are required to cause overflow on ‘bytesNeeded’.
  - length of ‘value’ in .crash file => 0 < {length of ‘value’} < 1024
  - length of ‘key’ in .crash file => UINT32_MAX - {length of ‘value’} - 7 < {length of ‘key’} < UINT32_MAX

```
/* Example; 'key' and 'value' have a length of 0xfffffffe and 1 respectively. */
$ python -c "print('A' * 0xfffffffe + ' : ' + 'B')" > /var/crash/fake.crash ## 0xfffffffe bytes length
$ cat fake.crash
AAA … AA : B
 ```

2) Trigger the apport daemon to read the fake.crash file
- Just create ‘fake.upload’ file by touch command.
- Or launch apport-gtk gui or apport-bug cli application.

3) Check out the result
- After a while, the daemon process has been killed by segmentation fault.


## Demo Video
[![Video Label](https://img.youtube.com/vi/OgIuHWeBQnU/0.jpg)](https://youtu.be/OgIuHWeBQnU)

