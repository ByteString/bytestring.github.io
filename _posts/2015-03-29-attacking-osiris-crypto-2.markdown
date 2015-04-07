---
layout: post
title:  "Attacking Osiris: Intercepting Handshake"
date:   2015-03-29 03:00:00
categories: security
---

This post follows on from yesterday's post which introduced the Osiris Combat Meter and a few related security concepts. In yesterday's post, I mentioned that after generating a random password, the security script within this meter distributes this password encrypted using a predefined key.

When I use the term 'encrypted' I am refering to a process where the data and the key are combined together to create a message which is hopefully unreadable to an attacker. In Osiris, this is done via XORing together the two pieces of data. I'll go into what XOR is in a moment but will mention once more than any attacks detailed against this meter rely on the assumption that a script is injected into the meter. This is a nontrivial task which has not yet been accomplished in the real world, so any attacks are theoretical and rely on conquering that obstacle.

XOR is a bitwise operator which in cryptography combines two values together in a reversible way. It is denoted by the symbol '⊕' but is often denoted as '^' also. A plaintext XORed with a key produces a ciphertext (or encrypted piece of text) and a ciphertext XORed with a key produces the plaintext again. Once the values have been XORed, the new value does not reveal any information about the original two.

It's worth noting that XOR has a bunch of mathematical properties but the most important is that it is commutative and associative, which means that if you're XORing a bunch of things together, you can mess with the order of operations in all manner of ways without impacting the end product.

Crucially, here are a few facts about XOR.

    Plaintext ⊕ Key = Ciphertext
    Ciphertext ⊕ Key = Plaintext
    AND: Plaintext ⊕ Ciphertext = Key

Because XOR happens at the level of individual ones and zeroes, these properties hold true for any segment of the strings also. The last two characters of the plaintext XORed with the last two characters of the key produce the last two characters of the ciphertext. At the binary level, XOR compares each row of bits and performs the following operation:

    10000100
    01101111⊕
    --------
    11101011

The end value is one if and only if the two values are different, otherwise the end value is zero.

In cryptography, a one time pad is a very secure method of encrypting data accomplished by XORing a piece of text with an equally long key. If the key is not equally long, it needs to be repeated as many times as it takes to be as long as the piece of text and this repetition can start to produce patterns within the ciphertext. For this method of encryption to be secure, it is crucial that the key is only used once and it's this reuse of the key that we'll be attacking today in Osiris.

From now on, P will be used to refer to Plaintext, K to the Key and C to the Ciphertext. These are not quite standard notation, but fuck it, I do what I want.

Given two plaintext values, P1 and P2, both encrypted using the key K we get C1 and C2 respectively. That is to say that:
    P1 ⊕ K = C1
    P2 ⊕ K = C2

We'll say that P1 is 11010011, P2 is 00101101 and K is 10100110.

So C1 is therefore:

    11010011
    10100110⊕
    --------
    01110101

While C2 is:

    00101101
    10100110⊕
    --------
    10001011

Now I'm not going to illustrate it here, you can do the working on your own, but C1 ⊕ C2 is 11111110. Interestingly, P1 ⊕ P2 is also 11111110. What we've identified here is that given the same key, C1 ⊕ C2 == P1 ⊕ P2. This is another important property of XOR operations. Given two ciphertexts, we can *entirely exclude the key from the equation* and deal purely with XORed plaintexts... but we'll come back to that.

If we look back at the previous post about the initalisation of the meter, we'll remember that each script shares knowledge of a predefined password which the security script uses in its first communication. This first communication transmits the 24 character key which the scripts use to encrypt their responses.

For the attacker with a script operating within the context of the meter, it is important to note that we have access to at least 19 ciphertexts encrypted using the same key. Even though this meter is open source, I'll work through this attack as if we know nothing about the internal communications.

Starting from being able to intercept encrypted communications, here is what we see when we output all the intercepted ciphertexts. The key being used is 'j@@thn%xb5o2i8sjmbfmp%^r':

    [01:21] RPCS Original: Ciphertext is: GSspGAQdWUlQAFgKXgtDWl1MVl1AFW5CX3RyRl1dFUhSG18CWQhDWhEeCQ4TSxY/Ajg0ID5ZHQEaZQFIOnE1Xg82BFQfGA==
    [01:21] RPCS Original: Ciphertext is: GCEuEw0KWU1VBl8CXwlDWl1MVl1AFW5CXHFyRFBdEUhSG18CWQhDWhEeFj82QGkCUjRvIQdBQUlWVg4HIEsWE1oPFhslGA==
    [01:21] RPCS Original: Ciphertext is: GSktCFtYFEBQB1wCWQhdWl1SVl1AFG5DXHZ3QlheFVZSBV8CWQgPFhwtBSNbaTAELXZ0HipdVg8tVFdZDVUxUjUuUlA=
    [01:21] RPCS Original: Ciphertext is: DiE0FQQBRBwHRxMKWg1EXllaVl1eFW5CWnBwRVhZEExSA18CWRZDWl1SVl0MWW8dIQYlJ14fVj8GfSpfImodRQpQNT4zSgQqGX0=
    [01:21] RPCS Original: Ciphertext is: HiEyEw0aTBYFSV0AUQxDXVhSVl1eFW5CWnBwRV1bHElWBV8CWRZDWl1SVl0MWRgDXgsCBkcsZEwqf1hAXQkgUykTMw4zFC8AL30=
    [01:21] RPCS Original: Ciphertext is: ByUsEQ0SHUxQAl4GWQhDWkNSVl1AFW5KWHl2QltZFUhSG18CWQhDWhEeHy4bEA0oIzgqFzEadgIAfQBABH4gWCVQLQkTGA==
    [01:21] RPCS Original: Ciphertext is: GCEtCF1dFElUDFgCWQhdWl1SVl1AF25DXHJ1QlheFVZSBV8CWQgPFl8pICwUaTwlPicWQV0BcRETGiQBIGAwX1kUJ1A=
    [01:21] RPCS Original: Ciphertext is: ByEpGhRaE01WAV0LWQhdWl1SVl1AEGxBWnhyQFheC0hSBV8CWXQwOiVJFh8HbBYDJARwIwkddjUkfApkD249I1A=
    [01:21] RPCS Original: Ciphertext is: ByU0ERoSHUtXBVoAWQhDRF1SVl1AFW9DW3J5QV5eFVZSBV8CWQgPFgkoNBs/bAwGQQcLRVwfYDEBARlTIgxLPzQlBVA=
    [01:21] RPCS Original: Ciphertext is: Di0nPAkAQRQHRxMAXg1FXV9SVl1AC25CWnBwRFBeFE9VBF0CWQhdWl1SVl1AWSIDCCgoBCUACjQVWSRfKls9Dy4kPlU9dDcjJRl9
    [01:21] RPCS Original: Ciphertext is: LQ08RlxaEkpQB18CWRZDWl1SVl1CHG5BWXV0RFheC0hSBV8CWUQPXA5bLQsIYxs8HA0OMFw2Sj8KDAlBHFclIjcnWw==
    [01:21] RPCS Original: Ciphertext is: Cy40HQsPSAgeBFoCXAlDXF1SVkNAFW5CWnB5RF9bEElRBV8CRwhDWl1SVhEMYBVKQQYsDVoJFzEIcxpiOXMFLgQqVDgZTRwFVw==
    [01:21] RPCS Original: Ciphertext is: CS8tGRsSFkpUBF8BUAhDWkNSVl1AFW5EX3B0QV5ZFUhSG18CWQhDWhEeKDceEjw8DQ9wDlotcEkMQVpnXGICEEYOBDoTGA==
    [01:21] RPCS Original: Ciphertext is: GiEzBx8BVxweAFkHXw5AXl1SVkNAFW5CWnB2QVBbHUhUBV8CRwhDWl1SVhEMbTQhBRgQAB8GSxEFW1d4DVQjDx0MIi41YG4RVw==
    [01:21] RPCS Original: Ciphertext is: KxAJCF1YEEpSBVoCWQhdWl1SVl1AFGZHXnZ2RFheFVZSBV8CWQgPFlkLAxpEEwkrMyctOBwCRxUBRxh2M0xLBgMjP1A=
    [01:21] RPCS Original: Ciphertext is: CSghBgkNUR0QSV0AWwxDWlxSVkNAFW5CWnB5RVlbFUpVBV8cWQhDWl1SGhEHHC4UQRoMRQEpTiIpQil8PwoFPxQpJx8YZC1P
    [01:21] RPCS Original: Ciphertext is: CSEwAB0cQARUBFYLXwlDWl1SSF1AFW5CWnl5QFlWE0hSBV8cWQhDWl1SGhEyEz8BIA8hETBBYAgOdF1GP08kIVkPCy8pdhNP
    
The first thing I see here is a series of similar segments which suggests a relationship between the encrypted messages. To be secure, these strings should not be mathematically related to one another, let alone visibly similar. See how at 23 characters in there's a segment of \"DRRRCVw\" with maybe a character's difference on either side? As an attacker, I don't know what this is but it suggests that a 'many time pad'attack exploiting reuse of key may be worthwhile.

We'll attempt a known plaintext attack to recover portions of the key and use these to decrypt portions of messages that do not contain known plaintext.

As an attacker, we can build a list of common words to try XORing against the ciphertext. In this example, I'll just jump straight to assuming that somewhere in the communications exists the word \"password\". It's a nice common security word and maybe the meter refers to it. We can do this automatically with a long list of likely words.

Readers will know what the attacker doesn't, that there is a message in the meter which has the plaintext value \"password\\|random\_number\\|password\_key\". If the attacker cycles through every ciphertext and XORs it with the string \"password\" repeated as many times as necessary, any ciphertexts which contains the word "password" reveal part the corresponding part of the key, where it lines up with the word \"password\". If \"password\" is not conventiently at the start of the string as it is here, we offset it to test for placement in different locations, trying the following:

    passwordpasswordpasswordpassword
    asswordpasswordpasswordpasswordp
    sswordpasswordpasswordpasswordpa
    swordpasswordpasswordpasswordpas

And so on. We'll stick to the start of the string for brevity but knowing the location of a keyword isn't required for this attack.

When we XOR a string starting \"password...\" with the ciphertexts, we receive a series of strings equally as nonsensical as the ciphertexts but if the assumption that 'password' is at the start of a string is correct, one of these strings starts with part of the key. If they key hadn't been recycled and was equal in length to the message to decrypt, we wouldn't be able to do anything with this information but knowing that the same key is in use we can construct a series of partial keys out of these results.

Because the word we're testing has eight characters, a successful match will give us eight characters of the key so we take all of the potential keys and keep the first eight characters.

    [01:30] RPCS Original: Potential key partial is: iJZksr+*
    [01:30] RPCS Original: Potential key partial is: wD_bz}b)
    [01:30] RPCS Original: Potential key partial is: h@^{.7g%
    [01:30] RPCS Original: Potential key partial is: [qz{&3e$
    [01:30] RPCS Original: Potential key partial is: ~LTO~o3p
    [01:30] RPCS Original: Potential key partial is: h@]`ze+)
    [01:30] RPCS Original: Potential key partial is: iH^{.8d+
    [01:30] RPCS Original: Potential key partial is: ~@Gfsn6x
    [01:30] RPCS Original: Potential key partial is: n@A`zu>r
    [01:30] RPCS Original: Potential key partial is: ]lO6*5o)
    [01:30] RPCS Original: Potential key partial is: {OGn\|`:l
    [01:30] RPCS Original: Potential key partial is: yN^jl}e(
    [01:30] RPCS Original: Potential key partial is: yIRu~b#y
    [01:30] RPCS Original: Potential key partial is: j@Asvb;y
    [01:30] RPCS Original: Potential key partial is: wDGbm}d)
    [01:30] RPCS Original: Potential key partial is: j@@thn%x
    [01:30] RPCS Original: Potential key partial is: w@Zic5g+
    [01:30] RPCS Original: Potential key partial is: y@Csjs2`
    
Now we just need to take a single ciphertext and try to use this partial key to decrypt it.

    [01:39] RPCS Original: Decryption attempt is: fmyG 0"8)M?M'yp*#???<x]2"=#?&188,I?M'+<(XhsFB(:VsZ!?yUy32?4O?u.eo using ~LTO~o3p
    [01:39] RPCS Original: Decryption attempt is: paph$::l?A?b#sh~5??=8rEf41*$"; l:E?b#!$\|NdziF""?eV(!}_g-%>??K?mzsc using h@]`ze+$
    [01:39] RPCS Original: Decryption attempt is: fajn-1'0)A?d*xu"#??;1yX:"10"+0=0,E?d**9 Xd`oO)?^sV2'tTzq3>??B?p&ec using ~@Gfsn6x
    [01:39] RPCS Original: Decryption attempt is: ram\|6140=A?v1xf"7??)*yK:617000.08E?v1** Ldg}T),^gV55oTiq'>??Y?c&qc using j@@thn%x
    
That last one is what we were looking for. If we pad "j@@thn%x" out with zeroes to make it the same length as the original key (this key length can be determined statistically or through trial and error for a real attacker) we get the following.
    
    [01:44] RPCS Original: Final decryption is: particle!yh2i8vohgdsp%^r00869064g2i,i8sjmb*!wA?,lqVbsNdg9\=S?>????$v?YV
    [01:44] RPCS Original: Final decryption is: main\|794b1m;i8mjmbfmp$_w1245000.b5o2i8????}-2b'
    qND0WasS?C?g?n???o
    [01:44] RPCS Original: Final decryption is: API\|2821`0k2i8mjmbfmp'_v1390000.b5o2i8?&i;3*t#9?YgmLtlbm1w(F?\|{63??`
    [01:44] RPCS Original: Final decryption is: dmgHandl7w#;j1zbjkfmp;^r00006173b1h2i8mjmbfmpi?3bhhpMn/L%i?o?k?????e?D??OY=
    [01:44] RPCS Original: Final decryption is: skills\|1`<j0k1sjm\|fmp%^r17375220b+o2i8sj!.9>#{&?hxtTV78y*U1x
    A?n??4d/(
    [01:44] RPCS Original: Final decryption is: sim\|3385k0h2i8mjmbfmp Wz3684000.b5o2i8?&,?5?kY?4G64jB3sw?dgi=e?b??b`
    [01:44] RPCS Original: Final decryption is: dataload7w#6j0ujdffmn%^r00084680a7o2w8sjmbf!<$?	FeS6qsGd?@2I?fl=o???/O61=
    [01:44] RPCS Original: Final decryption is: targetin5yn4h9uilbfmn%^r00078175a6o2i&sjmbfm<i(34KBr/BA4?Ohpm9?c?#?>?$?0E=
    [01:44] RPCS Original: Final decryption is: GM\|40555`5o2i&sjmbfmw-Xv705000.0b5o2it?l>k?;8S+?vMND4XoG:<9q,g????k
    [01:44] RPCS Original: Final decryption is: comms\|73`1f0i8sjsbfmp%^q51671000b+o2i8sj!.??."??gO0z2CU1<qjWlR2 v>4
    #(
    [01:44] RPCS Original: Final decryption is: password.2o;i<qnmbfsp%^r00838192k5o2w8sjmbf!<]??oXPtwhni5kgH=d??-<???P^!=
    [01:44] RPCS Original: Final decryption is: ranged\|2d5g3h0sjm\|fmp%^r12310180b5q2i8sjm.*-?S?up8t/Uo/dc1<clA0?$e;-6@S
    [01:44] RPCS Original: Key stealing attempt is: wDGbm}a-%m*x&g1 -3%.7z?/-??3.1g2"d,q.g}ryIGhH?~b1fx6+p?Uq`j Uc9[DDv#
    [01:44] RPCS Original: Final decryption is: melee\|48k1o0l8sjsbfmp%^w08281500\|5o2i8s&!+?6uF4?xjcYtSzb?j-o?[q?o?2>}
    [01:44] RPCS Original: Final decryption is: characte yi1a;pjdbfsp%^r00119987j5o2w8sjmbf!<bW2f+ZL1iGk?N(D?^q,?+??2}/1=
    [01:44] RPCS Original: Final decryption is: ram\|4558e1i2i8mjmbfmp#^{4093000.b5o2i8?&o???$Y??TgV55oTi#*?1?P?oi$?`
    [01:44] RPCS Original: Final decryption is: particle!yn3o8rkifeon%^r00017017d6m2w8sjmbf!<":,nlqVbsNd5n?`?Iu????/kO"z=
    
Guessing that "characte" means "character" we can do the entire process over again using "character" instead of "password". We can also see that the pipe character "\|" is being used to separate strings, so our test string becomes "character\|". Next we see that random numbers fill the middle, so for each position we cycle through every possible digit and see which ones make the most sense when the key partials they result in are compared against every ciphertext.

Another option available to us is to return to our assertion that C1 ⊕ C2 == P1 ⊕ P2. With more complex many-time pad examples, we can use this fact to fully decrypt ciphertexts statistical knowledge of common characters as well as information about how text is encoded when it goes through the encryption process.

The options for statistical cryptanalysis against a many-time pad are interesting but beyond the scope for this post, so hopefully I've demonstrated that an attacker with a script inside your shit is going to mess it up, which is my primary point.

More secure cryptography has a large overhead as far as memory is concerned and will lag your things to hell and even then there are further cryptographic attacks which can be used to tamper with data even where crypto prevents you from reading it.

This post has been rewritten a week or so after the actual experiments, so hit me up if it doesn't make sense in some parts as I probably missed old and nonsensical details.