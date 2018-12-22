# I want that toy, pwn
First we get a binary(server) and a libc, when reversing its clear that the program binds to a port, forks and then the parent closes
the fd, but the child serves the request. 
