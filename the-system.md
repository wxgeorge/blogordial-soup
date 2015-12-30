Operating systems was one of the courses during undergrad that revealed many mysteries of the machine to me. The course was University of Waterloo's CS 350. The elements of the course will resonate with recent students of the class, as well as students from many years ago. In my course we hacked on NachOS, a simple operating system with a few more features than the bare bones, but many empty holes to fill in. The course packaged it for us in a way that I appreciate now, of giving a virtual machine within which the operation system could executed and debugged. For the assignments, we had to implement basic functionality for components such as the filesystem or implementing virtual memory.

Working out the details of these components is valuable and important experience. You learn about software engineering. You learn about architecting for testing. You learn about important details of the system. You learn essential abstractions from the worlds of computing - processes, threads, file-system. And this resolved a ton of magic for me in how a system is oriented.

As I've spent more time at the project of software, I've learned more about what I would call 'the system'. One thing I've learned is there are a ton of _conventions_. Things that aren't exactly natural in the "this is the only way it could be" kind of way; our sisters and brothers on Alpha Centauri might have developed systems that look rather different overall; but our conventions, taken together, have a harmony and a logic, such that each individual convention is someone, just right.

ENVIRONMENT_VARIABLES

One of the important facilities of the system are environment variables. Environment variables give processes a means of communicating information among themselves. Processes have a variety of means of communicating amongst themselves (the filesystem is one way, directly interrogating another process through a socket is another); environment variables offer a primitive way of communicating information, which often employed to communicate some basic information used to bootstrap other parts of a pipeline.

Lets begin with what environment variables are (their syntax) and how you and your programs interact with them (system calls and common programs for interrogating them).

Environment variables are string-valued variables associated with a process. They are inherited from the environment that created the process (the `fork` system call on POSIX, the `spawn` family of calls on Windows). They may be modified by the process itself (for example, to pass some information on to a spawned process).

That's the entirety of the mechanics. You could get this from a manpage, but that's probably only helpful if you know why they're there and what they're typically used for. I want to give you my understanding of how they fit into the bigger picture of coordinating different programs that need to operate within _the system_.

If you have a POSIX terminal open, you can interrogate what environment variables are set therein with the command `env`. You can interrogate the value of the environment variables whose name is PATH with `echo $PATH`. By convention, we give environment variables names that are all uppercase. (hey, don't we usually write GET and POST in all caps? hmmmm ....). (I don't know the equivalent of `env` is for a `cmd` prompt, but `echo %PATH%` is the equivalent of `echo $PATH`). If your terminal is bash, you can set the value of the environment variable with the name FOO to have the value "bar" with the following commmend:

```
> export FOO=bar
```

(This may well work in other command interpreters like ZSH; I don't know why another command interpreter would bother change this syntax, but they are free to do so). The directive `export` is critical here. If we had simply written

```
> FOO=bar
```

FOO wouldn't show up in the output of env, though the result of 

```
> echo $FOO
```

would still be `bar`. (bash also has a notion of variable local to the command interpreter instance. This is useful when writing scripts, where one part of the script wants to communicate with another. Environment variables exist to communicate information across processes in a particular way).

Now some exercises:

1) We've shown how to query and set environment variables in a bash session. Write a clone of `env` as demoed above (i.e. to print all environment variables) in the language of your choice. Hint if your language of choice is C: consult the man pages for `environ` and `getenv`.

2) Suppose your `.bashrc` file contains a statement of the form `export PATH=$PATH:/Applications/CMake/Contents/bin` (or some other setting of an environment variable). If this file is executable, you'll be able to execute `> ~/.bashrc` but this won't have the effect of modifying your active session. Explain, and give the command that will modify the environment of the active session.

Basic Examples: HOME, USER

Lets consider how your command interpreter (e.g. bash, zsh) knows to expand `~` into your home directory? Suppose it knows your user name (learning that is a lower turtle). Suppose that was communicated to it through the USER environment variable (that probably would be a good idea). If all users in your system had their home directories at "/Users/$USER", then your command interpreter could infer the value of the home directory from the user name. But the layout of apple systems is different than, say, Ubuntu. On Ubuntu, the home directory is "/home/$USER". We could be running the same command interpreter in both operating systems, and they have to know the correct location in each.

One way to solve this problem would be that for bash, there's a code path that looks something like this:

```c
const char* get_user_home_directory() {
	const char* username;
	username = getenv("USER");
	if(NULL == username) { return NULL; }
	const char* prefix;
#if defined(APPLE)
	prefix = "/Users";
#else defined(UBUNTU)
	prefix = "/home";
#endif

	int home_length = strlen(prefix) + 1 /* sep */ + strlen(username) + 1 /* terminating null-character*/;
	char* home = malloc(sizeof(char) * home_length);

	int offset = 0;
	strcpy(home_length, prefix,   offset); offset += strlen(prefix);
	strcpy(home_length, "/",      offset); offset += strlen("/");
	strcpy(home_length, username, offset); offset += strlen(username);
	home[home_length-1] = 0;
	return home;
}
```

This is gross for the coupling it introduces. While you do need to know something about what environment you'll be executing in for compilation to be well defined (e.g. what architecture will we execute upon?), this is far, more coupling than we need. Even if we changed the conditionally compiled block to something that's deferred to run time evaluation (giving us a binary that could conceivably be ported from one operating system to another), we still have bizarre and gross coupling between our command-interpreter and the entire zoo of possible operating systems and operating system layouts. It's gross if we can't change something like where we write the users home directories because it's baked into a multitude of basic programs like bash. Moreover, this logic would need to be duplicated across _every command interpreter_. This is awful, and this pain would not be worth convenience we get .

The answer of how we do it, is the operating system promises to set the HOME environment variable to the home directory of the current user. This is one of the requirements for being a POSIX compliant operating system (TODO: check if Windows does the same).

So I claim, without having looked at the source code for bash and zsh, that these command interpreters know how to expand ~ based on the $HOME environment variable. We can test this by updating the value of HOME and seeing how the shell behaves. That's an exercise. Go do it. So we have an example of how to take a basic mechanism for communicating information, and are using it to coordinate a particular program's execution within the broader system.

Now that you've seen this change we observe a significant point here: we were able to modify the behaviour of a compiled program, without recompiling it's source code or changing it's configuration files. This is an important point. This kind of flexibility is flexibility that all programs need in some respect or another. Sometimes this flexibility should arise through configuration files. Sometimes this flexibility is best provided with command-line switches provided at invocation time. And sometimes this flexibility is best provided with environment variables.


Examples Con't: PATH

Let's look at another environment variable with (arguably) more significant semantics than our previous example: PATH.

So: you open a terminal and type python. We're doing this because we want to run an instance of the python interpreter. Now python comes pre-installed in OSX and Ubuntu, so maybe it took Apple and Canonical an great deal of work of customizing bash so that when I type `python`, it runs. You don't think they did that?

People who have installed python on Windows may have experienced some of this pain. They downloaded a file from the internet that started with python and ended with `.msi`. They double-clicked that file, and they got some icons on their desktop and in the start menu, but if they open a `cmd` prompt and type python, nothing happens. What gives?

POSIX users will be familiar with the program `which`. (man which! go make me a sandwich!). `> which python` will result in the output "/usr/bin/python" for both OSX and Ubuntu (/usr/bin is the expected location of all the programs installed by default in the system). `ls`ing that path reveals that file exists. And if you run `> /usr/bin/python` this has the same effect as `> python`. `/usr/bin/python` is the fully-qualified path of the python interpreter in your file-system. So how did bash find it?

The answer concerns the convention of PATH: what it specifices, and how it is formatted.

PATH is interpretted by bash (and many other programs) as a colon-seperated list of strings, each interpretted as a directory that could contain programs. When you execute `> python`, to resolve this command to a fully-qualified path, a bash subroutine will go through this list of directories from left to right, and stopping when it first locates a program called python, and output that fully-qualified path.

Let me sketch that out. I'll do python, since there's more string manipulation than I care to do in C.

```python
def resolve(progam):
	path = os.env["PATH"]
	for dir in path.split(":"):
		candidate = dir + "/" + program
		if os.path.exists(candidate): return candidate

	return None
```

This is an important point. This is an incredibly important mechanism, understanding of which gives you new control of your system.

People that work on the command-line all the time understand what PATH is and how to use it to there advantage. I have a `bin` folder under my home directory that I throw programs in there for which there isn't a package in the Ubuntu repositories or the homebrew packages (e.g. zebra crossing)

Examples Con't: CLASSPATH, PYTHONPATH, GOPATH, GEM_PATH

Similar to the problem of "where should a command interpreter look for installed program?", there is the problem of

	 -"where should your java virtual machine look for compiled java code when resolving import statements?"
	 -"where should your python interpreter look for modules when resolving import statements?"

Ditto for go, for ruby, for javascript and so on.

Again, since the answer is system dependent, we need a convention by which to communicate information about the environment in which these programs are running. Again, we agree to set a particular environment variable. When you run a java virtual machine in an environment where the environment variable CLASSPATH is defined, this will be interpreted with the same syntax as the PATH variable (a colon separated list of paths), but these locations will be searched first for java .class files when resolving import statements.
