---
layout: post
title: Weakness of the One-Time Pad
description: "Why you never use the same OTP key twice."
modified: 2014-1-19
tags: [otp, crypto, coursera]
image:
  feature: top-background.jpg
  credit: chrisballinger
  creditlink: http://www.flickr.com/photos/chrisballinger/11314777016/in/set-72157638559926193
comments: false
share: true
category: blog
---

I've been taking Coursera's [Stanford Cryptography I](https://class.coursera.org/crypto-009) class and last week's homework had an interesting extra credit problem: given eleven ciphertexts all encrypted with the same [one-time pad](https://en.wikipedia.org/wiki/One-time_pad) key, find the plaintext of the last ciphertext. It included a hint to [XOR](https://en.wikipedia.org/wiki/XOR_cipher) the ciphertexts together and to take a note of what happens when you XOR a space with lowercase and uppercase characters.

When I initially went to solve the problem, I thought that it would make the most sense to use [frequency analysis](https://en.wikipedia.org/wiki/Frequency_analysis) but hit a dead-end after a few hours and felt pretty frustrated. However, after studying the hint more carefully, I realized:

    'ABC' XOR '   ' = 'abc'
    'abc' XOR '   ' = 'ABC'

Eureka! If you combine that with these properties of XOR:

    'plain text' XOR 'secretkeys' = 0x0309021b0b541f000107
    'safe texts' XOR 'secretkeys' = 0x0004051745000e1d0d00
    
    'plain text' XOR 0x0309021b0b541f000107 = 'secretkeys'
	'safe texts' XOR 0x0004051745000e1d0d00 = 'secretkeys'
	
	0x030d070c4e54111d0c07 = 'plain text' XOR 'safe texts'
	0x030d070c4e54111d0c07 = 0x0309021b0b541f000107 XOR 0x0004051745000e1d0d00
	
Pay close attention to the last two lines. You'll notice that the XOR of two ciphertexts encoded with the same OTP key is identical to the XOR of the two plaintexts. Now, let's look closer at the contents of `0x030d070c4e54111d0c07`:
	
    ['\x03', '\r', '\x07', '\x0c', 'N', 'T', '\x11', '\x1d', '\x0c', '\x07']

You'll notice the uppercase 'N' and 'T' characters definitely stand out. From what we learned earlier, this means that the 5th position of the two original plaintexts has a 50% chance of being either a space or a lowercase 'n', and the 6th position is either a 't' or a space.

I used this knowledge, along with the XOR permutation of the 11 given ciphertexts to assemble the probability of each character at a given 

https://gist.github.com/chrisballinger/8515411