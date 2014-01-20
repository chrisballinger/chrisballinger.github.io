---
layout: post
title: Weakness of the One-Time Pad
description: "Why you never use the same OTP key twice."
modified: 2014-1-19
tags: [otp, crypto, coursera, xor]
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

** Spoiler Alert: If you're currently taking this class and haven't completed this homework, please do not read any further until you have solved it on your own. Your solution will likely be better than mine. **

I used this knowledge, along with the XOR permutation of the 11 given ciphertexts with each other to assemble the probability of each character at a given index. Basically, I had a parallel 'probability' array that kept track of the chance of each character occurring at each index. 

{% highlight python %}
def scan_string(string, probability_arrays, ct1_index, ct2_index):
    characters = list(string)
    
    for character_index in range(len(string)):
        character = characters[character_index]
        letter = scan_for_letter(character)
        if letter:
            set_letter_at_index(letter, character_index, probability_arrays, ct1_index, ct2_index)
            
def permute_array(cipher_texts, probability_arrays):
    for x in range(len(cipher_texts)):
        for y in range(x+1, len(cipher_texts)):
            xor = xor_indices(cipher_texts, x, y)
            scan_string(xor, probability_arrays, x, y)
{% endhighlight %}

The `scan_for_letter` function checks if a given character is within [a-zA-Z] and returns the swapped case of that letter if found. The `set_letter_at_index` function helps us keep track of the occurance of each letter by recording both a space and the letter in two probability arrays, each array corresponding to the one of the two currently XORed ciphertext pairs.

It ends up looking something like this:

    1. [{'T': 3, ' ': 3}, {'h': 3, ' ': 3}, {'e': 3, ' ': 3}, {' ': 2, 'e': 2, 'r': 2}, {}, ...]
    ...
    
I figured that if for a given index, a non-space character with a 0.5 probability was probably that character. For a short-cut, basically all 2-element letter probability dictionaries could be assumed to be whatever non-space character was present. Otherwise, if there is no clear letter match, it is probably a space character. Sometimes there are no letter matches at all so I output a '`*`' instead, with the final output looking something like this:

	1. The *...

{% highlight python %}
def most_probable(prob_dict):
    if len(prob_dict) is 0:
        return ' '
    total_character_count = 0
    keys = prob_dict.keys()
    for key in keys:
        character_count = prob_dict[key]
        total_character_count = total_character_count + character_count
    for key in keys:
        prob_dict[key] = float(prob_dict[key]) / total_character_count
    highest_probability_index = 0
    for i in range(1, len(keys)):
        key = keys[i]
        probability = prob_dict[key]
        highest_probability = prob_dict[keys[highest_probability_index]]
        if probability > highest_probability:
            highest_probability_index = i
    most_probable_key = keys[highest_probability_index]
    return most_probable_key
{% endhighlight %}

The `most_probable` function probably isn't the most efficient or beautiful way of calculating this, but it is very useful and I end up reusing it later when refining the key guesses. You could have stopped right here and had a reasonable guess for each original plaintext, missing some characters here and there.

However, I thought that I could improve the accuracy of my plaintexts by guessing the key that the most generated plaintexts had in common with each other. I re-used the `most_probable` function to find the most common key by keeping track of the key bytes at each character index and finding the most commonly shared key. This allowed me to find the non-letter characters that were left, like punctuation and even the sneaky single-character '`ﬁ`' present in one of the plaintexts.

{% highlight python %}
def guess_key(guess_texts, cipher_texts):
    key_guess_probabilities = generate_key_guess_array(length_of_longest_array(guess_texts))
 
    for i in range(len(guess_texts)):
        guess_text = guess_texts[i]
        key_guess = strxor(cipher_texts[i].decode("hex"), guess_text)
        update_key_probabilities(key_guess, key_guess_probabilities)
 
    key_guess = guess_key_from_probabilities(key_guess_probabilities)
    return key_guess
{% endhighlight %}


Unfortunately my solution still has bugs (can you find them?) and I wasn't able to completely automate the key-finding process. There were still a bunch of garbage characters that I needed to guess based on context clues. I ended up repeatedly running the `guess_key` function on manually refined plaintexts until I found the original OTP key that decrypts all of the ciphertexts in their entirety.

I could have automated the process a bit more by using an English dictionary and guessing the missing characters within words using the [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance) to known words, but I don't know how well that would work. I'm still interested in a completely automated solution that doesn't require any manual intervention but, for now, this will have to do.

Remember, never use the same OTP key more than once or your ciphertexts will leak information about the original plaintexts and eventually reveal your key.

If you're interested in the rest of the source code, it's available as [redacted Gist](https://gist.github.com/chrisballinger/8515411) on GitHub.