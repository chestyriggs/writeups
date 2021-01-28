#!/usr/bin/env python

from pwn import *

TCP_IP = 'fibonacci'
TCP_PORT = 7600

FINAL_NUMBER = 0

######## A quick and dirty fibonacci cruncher #######
# It will just throw an error when we get the eventual flag. Should probably do a
# check if the returned value is a new fib number or an actual flag and exit
# accordingly
####################################################


def reverseFibonacci(fib):
    # Had to add this in to compensate for the last number (was missing)
    n = fib + 1
    a = [0] * n

    # First and second elements
    a[0] = 0
    a[1] = 1

    # Storing the sum in the preceding location
    for i in range(2, n):
        a[i] = a[i - 2] + a[i - 1]

    return a[-1]


# Connect to the server
with remote(TCP_IP, TCP_PORT) as r:

    while True:
        # Get the F-number
        data = r.recv()
        log.info(f"> Recieved Message: {str(data, 'utf-8')}")

        # Convert the data we got into an Int
        n = int(data.decode("utf-8").split("(")[1].split(")")[0])

        # Get the value for Fn
        val = reverseFibonacci(n)
        log.info(f"> fib({n}): {val}")

        # Send the value for Fn
        log.info("> Sending value...")
        r.send(str(val))

    # Close the connection
    r.close()
