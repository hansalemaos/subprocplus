# subprocess + some useful stuff 

### Tested against Windows / Python 3.11 / Anaconda

### pip install subprocplus

```py
# Some nice things to do with subprocess
# Includes stuff from:
# https://github.com/zeroSteiner/mayhem
# https://stackoverflow.com/a/29737399/15096247
# https://github.com/hansalemaos/procciao

from subprocplus.subproc import (
    create_new_user,
    disable_internet_for_user,
    remove_user,
    CREATIONINFO,
    Popen,
    PIPE,
    time,
    CREATION_TYPE_LOGON,
    subprocess,
    remove_firewall_rules,
)
import sys


username = "NO11"
password = "NOINTERNET"
# removing a user
remove_user(username)
time.sleep(5)
# creating a new user
usern, passw = create_new_user(new_username=username, new_password=password, admin=True)
time.sleep(5)

# disable internet usage for apps for created new user, returns a list of created rules
rules = disable_internet_for_user(
    username=username,
    password=password,
    apps=[r"C:\Windows\System32\curl.exe", r"C:\msys64\usr\bin\wget.exe"],
    new_display_name1=None,
    new_display_name2=None,
)
time.sleep(5)
ci = CREATIONINFO(
    CREATION_TYPE_LOGON,  # CREATIONINFO for logged on user (from  https://stackoverflow.com/a/29737399/15096247 )
    lpUsername=username,
    lpPassword=password,
    dwCreationFlags=subprocess.CREATE_NO_WINDOW,  # invisible
    dwLogonFlags=1,
)
cmd = "powershell.exe"
p1 = Popen(
    cmd,
    suspended=False,
    creationinfo=ci,
    stdout=PIPE,
    stderr=PIPE,
    stdin=PIPE,
    print_stdout=True,
    print_stderr=True,
)  # starting the subprocess as a logged on user

print("Process Id: %d" % p1.pid)
print("Thread Id: %d" % p1._processinfo.dwThreadId)
p1.sendcommand(
    "ls",
    clean_old=True,  # all results are saved in p1.out_dict and p2.err_dict
    restart_on_fail=True,  # restarts if there is a broken pipe or something else
    max_restarts=3,
    sleep_after_restart=10,
)
time.sleep(5)
print(
    p1.get_last_stdout(clean=True)
)  # converts p1.out_dict to a list and clears p1.out_dict
print(
    p1.get_last_stderr(clean=True)
)  # converts p1.err_dict to a list and clears p1.err_dict
p1.sendcommand(
    r"C:\Windows\System32\curl.exe google.com"
)  # connection error, because it is blocked
time.sleep(3)
p1.sendcommand(
    r"C:\msys64\usr\bin\wget.exe google.com"
)  # connection error, because it is blocked
time.sleep(3)
remove_firewall_rules(rules)  # deleting the created firewall rules
time.sleep(5)
p1.sendcommand(r"C:\Windows\System32\curl.exe google.com")  # no more connection error
time.sleep(3)
p1.sendcommand(r"C:\msys64\usr\bin\wget.exe google.com")  # no more connection error
time.sleep(3)
print(p1.get_last_stdout(clean=True))
print(p1.get_last_stderr(clean=True))
p1.stdin.close()  # provoking an error, because it is closed
time.sleep(5)
p1.sendcommand(
    "dir",
    clean_old=True,
    restart_on_fail=True,
    max_restarts=3,
    sleep_after_restart=10,
)  # reconnects after error
time.sleep(5)
print(p1.get_last_stdout(clean=True))

ci1 = CREATIONINFO(
    CREATION_TYPE_LOGON,
    lpUsername=username,
    lpPassword=password,
    dwCreationFlags=subprocess.CREATE_NO_WINDOW,
    dwLogonFlags=1,
)
p2 = Popen(
    r"C:\Windows\System32\cmd.exe",
    suspended=True,  # doesn't start
    creationinfo=ci1,
    stdout=PIPE,
    stderr=PIPE,
    stdin=PIPE,
)
print("Process Id: %d" % p2.pid)
print("Thread Id: %d" % p2._processinfo.dwThreadId)
assert not p2._child_started
input("Press enter to start")
p2.start()  # there we go
assert p2._child_started

time.sleep(5)
# https://github.com/zeroSteiner/mayhem
for key, item in p2.maps.items():
    try:
        # reads the memory. If NumPy is installed, it will be much faster,
        # because it uses NumPy's buffer protocol.
        # Be careful when using it without NumPy, it might print forever.
        mymem = p2.read_memory(key, item.size).view("V1").view("S1")
        print(mymem)
    except Exception as e:  # some protected areas can't be read
        sys.stderr.write(f"{e}")
        sys.stderr.flush()
```