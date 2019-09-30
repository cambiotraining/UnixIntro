---
title: "Using remote resources"
teaching: 15
exercises: 15
questions:
- "How can I work on the unix shell of a remote computer:"
- "How can I move files between computers"
- "How can I access web resources using the command line"
- "Mounting a directory from another computer onto your local filesystem"
- "When might different remote access tools be more appropriate for a task"
keypoints:
- "`ssh` a secure method of running a bash terminal on a remote computer on which you have a user"
- "`scp` a method of copying files using the ssh protocol"
- "`sshfs` a method of using the ssh protocol to connect your local filesystem directly to a remote filesystem as if they were connnected"
- "`rsync` a different method of file transfer which only copies files that have changed"
- "`wget` a command which can download files from http links as if it were in a browser"
---
Let’s take a closer look at what happens when we use the shell on a desktop or laptop computer. The first step is to log in so that the operating system knows who we are and what we’re allowed to do. We do this by typing our username and password; the operating system checks those values against its records, and if they match, runs a shell for us.

As we type commands, the 1’s and 0’s that represent the characters we’re typing are sent from the keyboard to the shell. The shell displays those characters on the screen to represent what we type, and then, if what we typed was a command, the shell executes it and displays its output (if any).

What if we want to run some commands on another machine, such as the server in the basement that manages our database of experimental results? To do this, we have to first log in to that machine. We call this a remote login.

In order for us to be able to login, the remote computer must be running a remote login server and we will run a client program that can talk to that server. The client program passes our login credentials to the remote login server and, if we are allowed to login, that server then runs a shell for us on the remote computer.

Once our local client is connected to the remote server, everything we type into the client is passed on, by the server, to the shell running on the remote computer. That remote shell runs those commands on our behalf, just as a local shell would, then sends back output, via the server, to our client, for our computer to display.

## The ssh protocol

SSH is a protocol which allows us to send secure encrypted information across an unsecured network, like the internet. The underlying protocol supports a number of commands we can use to move information of different types in different ways. The simplest and most straightforward is the `ssh` command which facilitates a remote login session connecting our local user and shell to any remote user we have permission to access.

~~~
$ ssh sshuser@127.0.0.1 -p 8890
~~~
{: .language-bash}

The first argument specifies the location of the remote machine (by IP address or a URL) as well as the user we want to connect to seperated by an `@` sign. For the purpose of this course we've set up a container on your local machine for you to connect to the IP address `127.0.0.1` is actually reserved for your local computer.

We also specify the port to look for the ssh server on with `-p 8890`. Most network based services listen for connections on a specific numbered port for ssh the default is 22. However, a common security measure is to change the port ssh is listening on to avoid opportunistic connections. In our case we are using 8890 to avoid conflict with the existing ssh server used on your computer for administration.

~~~
The authenticity of host '[127.0.0.1]:8890 ([127.0.0.1]:8890)' can't be established.
ECDSA key fingerprint is SHA256:v9X5DCaGtI0mSF79Krmhx3g8AbMQqVAhg6hHEjdexho.
Are you sure you want to continue connecting (yes/no)? yes
~~~
{: .output}

When you connect to a computer for the first time you should see a warning like the one above. This signifies that the computer is trying to prove it's identity by sending a fingerprint which relates to a key that only it knows. Depending on the security of the server you are connecting to they might distribute the fingerprint ahead of time for you to compare and advise you to double check it in case it changes at a later log on. In our case it is safe to type `yes` .

~~~
sshuser@127.0.0.1's password: ********
~~~
{: .language-bash}

Now you are prompted for a password. In an example of terribly bad practice our password is the same as our username `sshuser` .

~~~
    sshuser@7a7882cd4d46:~$
~~~
{: .language-bash}

You should now have a prompt very similar to the one you started with but with a new username and computer hostname. Take a look around with the `ls` command and you should see that your new session has its own completely independent filesystem. Unfortunately it's rather empty. Let's change that, but first we need to go back to our original computer's shell. User `Ctrl+D` on an empty command prompt to log out.

~~~
$ ls -la /home
~~~
{: .language-bash}

## Moving files

`ssh` has a simple file copying counterpart called `scp` which uses all the same methods for authentication and encryption but focuses on moving files between computers in a similar manner to the `cp` command we learnt about before.

Making sure we're in the `data-shell` directory let's copy the notes.txt file to the remote machine

~~~
$ cd ~/Desktop/data-shell
$ scp -P 8890 notes.txt sshuser@127.0.0.1:/home/sshuser
~~~
{: .language-bash}

The format of the command should be quite familiar when comparing to the `cp` command for local copying. The last two arguments specify the source and the destination of the copy respectively. The difference comes in that any remote locations involved in the copy must be preceded by the `username@IP` syntax used in the `ssh` command previously. The first half tells scp how to access the computer and the second half tells it where in the filesystem to operate, these two segments are separated by a `:` .

Note that scp uses a capital `-P` to specify the port number

~~~
$ ls lengths.txt
~~~
{: .language-bash}

~~~
notes.txt                                     100%   86   152.8KB/s   00:00
~~~
{: .output}

Now it looks like we've copied the file, but we should check.

Establishing a whole ssh session just to run one command might be a bit cumbersome. Instead we can tell ssh all the commands it needs to run at the same time we connect by adding an extra argument to the end. Ssh will automatically disconnect after it completes the full command string.

~~~
$ ssh -p 8890 sshuser@127.0.0.1 "ls /home/sshuser"
~~~
{: .language-bash}

~~~
notes.txt
~~~
{: .output}

Success!

> ## How do we get files back from the remote server?
>
> Using two commands we've uploaded a file to our server and checked it was there
>
> ~~~
> $ scp -P 8890 notes.txt sshuser@127.0.0.1:/home/sshuser
> $ ssh -p 8890 sshuser@127.0.0.1 "ls /home/sshuser"
> ~~~
> {: .language-bash}
> 
> More often we might want to have the server do some work on our files and then get them back. We can use exactly the same syntax structure with different arguments. 
>
> Try and make a change to notes.txt (perhaps add some text to the end with `echo` and `>>`) in the remote location and then retrieve the changed file. Remember to change the name of the file as you already have a notes.txt in your directory.
>
> If you're having trouble check `man scp` for more information about how to structure the command
>
> > ## Solution
> > ~~~
> > ssh -p 8890 sshuser@127.0.0.1 "echo all done >> notes.txt"
> > scp -P 8890 sshuser@127.0.0.1:notes.txt changed_notes.txt
> > ~~~
> {: .solution}
{: .challenge}

##Managing multiple remote files

Sometimes we need to manage a large number of files across two locations, often maintaining a specific directory structure. `scp` can handle this with it's `-r` flag. Just like `cp` this puts the command in recursive mode allowing it to copy entire directories of files

~~~
$ scp -r -P 8890 workflow/ sshuser@127.0.0.1:/home/sshuser
~~~
{: .language-bash}

~~~
.run_alignment.sh.swp                         100%   12KB   7.5MB/s   00:00    
ecoli_rel606.fasta.amb                        100%   12    12.0KB/s   00:00    
ecoli_rel606.fasta.rbwt                       100% 1696KB 162.6MB/s   00:00    
ecoli_rel606.fasta.rpac                       100% 1130KB 312.5MB/s   00:00    
ecoli_rel606.fasta.ann                        100%   87   431.4KB/s   00:00    
ecoli_rel606.fasta.bwt                        100% 1696KB 325.9MB/s   00:00    
ecoli_rel606.fasta.rsa                        100%  565KB 289.3MB/s   00:00    
ecoli_rel606.fasta                            100% 4578KB 331.4MB/s   00:00    
ecoli_rel606.fasta.sa                         100%  565KB 294.5MB/s   00:00    
ecoli_rel606.fasta.pac                        100% 1130KB 309.9MB/s   00:00    
ecoli_rel606.fasta.fai                        100%   29    78.3KB/s   00:00    
run_alignment.sh                              100% 1143     4.1MB/s   00:00    
SRR000011.fastq                               100%  128KB 223.8MB/s   00:00    
SRR000013.fastq                               100%  128KB 273.4MB/s   00:00    
SRR000016.fastq                               100%  140KB 188.2MB/s   00:00    
SRR000012.fastq                               100%  128KB 162.7MB/s   00:00    
SRR000015.fastq                               100%  128KB 170.5MB/s   00:00    
SRR000014.fastq                               100%  139KB 226.5MB/s   00:00 
~~~
{: .output}

However `scp` isn't always the best tool to use for managing this kind of operation.

When you run `scp` it copies the entirety of every single file you specify. For an initial copy this is probably what you want, but if you change only a few files and want to synchronize the server copy to keep up with your changes it wouldn't make sense to copy the entire directory structure again.

For this scenario `rsync` can be an excellent tool.

## rsync

First lets add some new files to our `workflow` directory using the `touch` command. This command does nothing but create an empty file or update the timestamp of an existing file.

~~~
$ touch workflow/newfile1 workflow/newfile2 workflow/newfile3
~~~
{: .language-bash}

`rsync` is already set up on our local computer, but as the remote server is a fresh install we'll need to add it (although you can generally expect it to be installed as default).

We need to first ssh to the remote server and then issue a command to add `rsync` as follows.

~~~
$ ssh -p 8890 sshuser@127.0.0.1
$ sudo apt -y install rsync
~~~
Finally Ctrl+D to exit.




## Wget - Accessing web resources












