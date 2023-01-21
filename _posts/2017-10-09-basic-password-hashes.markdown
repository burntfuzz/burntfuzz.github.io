---
layout:     single
classes:    wide
title:      "Basics - Password Hashes"
date:       2017-10-09 12:00:00
categories: ctf basics
---

<p>This Basics post will give an overview of password hashes and the process of password cracking. I thought it would be
worthwhile to discuss the basic principles before jumping into a tutorial for using any specific tool. This post will cover the basics of
hashing and types of password cracking methods.</p>


<h2 class="section-heading">Hashing Basics</h2>
<p>As a general rule, passwords should never be stored in plaintext. Typically, they are run through some hashing algorithm and stored as hashes.
The goal of a hashing algorithm is to take data of any size and reduce it to a fixed-length string of characters. You can think of a hash as a signature for that particular data. Here are some essential features of cryptographic hash functions: </p>


<ul>
	<li>A hashing function should be resistant to collisions; Ideally, one set of data should never result in the same hash as another set of data.</li>
	
	<li>Hashes cannot be reversible. Given the hash, it should not be possible to generate the data that resulted in that hash. A hash should be like a baked cake to a set of ingredients; There shouldn't be a way to get the flour back out of that cake. </li>
						
	<li>If the data is altered in any way, it should result in a completely different hash. As an example, lets compare the MD-5 hashed versions of
	two passwords: "password" and "passw0rd". These two strings differ by only one character but should result in completely different hashes. You can confirm these yourself using an MD-5 hash generator like <a href='http://www.miraclesalad.com/webtools/md5.php' target="_blank">this one</a>.

			<ul style="padding-left:70px" style="padding-top: 30px">
				<li>"password" results in this MD-5 hash: <code class="codespan">5f4dcc3b5aa765d61d8327deb882cf99</code></li>

				<li>"passw0rd" results in this MD-5 hash: <code class="codespan">bed128365216c019988915ed3add75fb</code></li>
			</ul>
</li>
</ul>


<p>While MD5 is still widely used, it should be noted it is not considered a secure cryptographic hash algorithm as it is prone to collisions. Two examples of widely used secure hashing algorithms are bcrypt and scrypt. Among other features, these algorithms internally apply a "salt", which is a randomly generated string that is added to the password before it is hashed. The salt is different for each password, and is generally not encrypted.</p>

<h3 class="section-heading">Salts</h3>
<p>
Salts are a very easy measure for a developer to employ that makes hashed passwords a little harder to crack. A salt can be added to a hash function as simply as <code>some_hash_function(password + salt)</code>. A salt is generally a randomly generated  unique string that doesn't necessarily have to be secret. For example, a username could work as a salt, as long as it is unique and does not change, but choosing something more complex is generally better. So how does this help secure passwords (further than just making the input to the hashing algorithm more complex)? Well, imagine if someone was cracking a list of hashed passwords, and there were duplicates of the same hash for different users. Since hash functions by definition will always generate the same result for the same input, it would be safe to assume those users had the same password. You get 2 users for the price of 1! Salts prevent this. If everyone has a unique salt, then everyone will also have a unique password hash even if they have the same password, so no one is getting any 2 for 1 deals.
</p>

<h2 class="section-heading">General Cracking Strategy</h2>

<p>There are a few different methods of password cracking that excel in discovering different types of passwords in shorter amounts of time. However, the general strategy between them is the same. Password cracking consists of two parts: locating or creating a file containing the hashed passwords, and then using some method to convert the hashed passwords into plaintext. This hash file can be a system file (Like SAM on Windows or etc/shadow on Linux) if you want an account password for a specific machine or user, or a text file containing hashes stored in a dumped database table. I'll talk more in depth about acquiring the hash files from a physical machine in another guide, so for now we'll just assume we already have one and are ready to start the cracking process. </p>



<p>The cracking process is completed through these steps:</p>
<ul>
	<li>Select a hashing algorithm (MD-5, SHA-256, SHA-512, etc.)</li>
	<li>Pick a plaintext word, encrypt using the algorithm</li>
	<li>Compare our created hash with the target's hash</li>
	<li>Pick a new plaintext word and repeat</li>
</ul>

<p>If our created hash matches a target hash, then we have discovered a password. As you can probably tell, this can potentially take a lot of guesses. Remember the analogy with the cake? We're not getting the ingredients out of the cake, we're trying to bake the exact same cake but by completely guessing the proportion of each ingredient thousands of times until we get it right. And with password hashes, there's no way to tell how close we are. The way the different cracking attacks differ is in the way the plaintext word is guessed. Making good guesses means the process will take less time.</p>


<h2 class="section-heading">Types of Password Cracking Attacks</h2>

<h3>Dictionary Attacks</h3>
<p>A dictionary attack makes use of a dictionary (AKA a wordlist) to guess passwords. These wordlists are text files populated with common passwords. Most of these are actual passwords acquired from various data breaches. The largest and most well known wordlist is the RockYou list, which contains 32.6 million stolen passwords. Taking a look at some of the <a href='https://www.passcape.com/index.php?section=blog&cmd=details&id=17' target="_blank">most-used passwords</a> in the RockYou list might be useful as a guideline for what not to use as a password. A dictionary attack should generally be the first approach to cracking a password, and can potentially succeed in minutes. Most of the password cracking you'll be doing for HTB and the PWK labs will be dictionary attacks, loading hashes into tools like JtR or hashcat and trying various wordlists. </p>

<h3>Rainbow Table Attacks</h3>
<p>A rainbow table attack is similar to a dictionary attack in that it guesses passwords from a preset list. A dictionary attack runs each word in a wordlist through a hashing function and gets the hash during the cracking process. However, a rainbow table attack involves a list of already-hashed passwords, which bypasses the need to calculate the hash for every word. Rainbow tables can save some time at the expense of storage space. It should also be noted that they're rather ineffective against salted passwords, since the attacker would need to make his own rainbow table to account for the unique salts, meaning no time would be saved over a regular dictionary attack.</p>

<h3>Brute Force Attacks</h3>
<p>Brute force attacks attempt all combinations of letters, numbers, and special characters. This is the most time-consuming method and should generally be a last resort. However, they can be effective against shorter passwords. Lets say that we're trying to crack a six-character password and we've had no success with our other cracking attempts. The total number of guesses we would need to make is 95^6 (95 is the number of printable ASCII characters) which comes out to 735,091,890,625 possibilities. Modern GPUs can make this amount of guesses in <a href='https://arstechnica.com/information-technology/2012/12/25-gpu-cluster-cracks-every-standard-windows-password-in-6-hours/'>a very short amount of time</a>, even if the attacker only has one or two of them. </p>

<h3>Hybrid Attacks</h3>
<p>A hybrid attack uses a combination of dictionary words, numbers, and special characters. These attacks follow a list of rules to determine how the numbers and special characters are incorporated into the guesses. For example, a rule might be to try replacing 'E' with '3' or 'O' with '0'. Other rules might append numbers or characters to the end of the password, or attempt to try concatenating one or more dictionary words. Hybrid attacks are typically attempted after running the hash file through a regular dictionary attack.</p>

Like I said earlier, most of what you'll be doing on HTB or the PWK labs are dictionary attacks with JtR or hashcat. Typically getting the password hash is the challenge itself and if it's meant to be cracked, it probably won't take that long. It's still useful to know the distinctions between the different types; and they seem like something that would show up on a certification exam like CEH or Security+.


