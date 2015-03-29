---
layout: post
title:  "Attacking Osiris: Intercepting Handshake"
date:   2015-03-29 03:00:00
categories: security
---

_UPDATE: Rereading this I am entertained by the considerable redundancy in the steps taken. In my defence, it was the wee hours of the morning and I was hoping to demonstrate both attacks that relied on XORing ciphertexts together and those that relied on known plaintext. I kind of combined them and it should be noted that for a known plaintext attack it's not necessary at all to XOR the ciphertexts together. I may rewrite this post entirely in a way that makes more sense._

This post follows on from yesterday's post which introduced the Osiris Combat Meter and a few related security concepts. In yesterday's post, I mentioned that after generating a random password, the security script within this meter distributes this password encrypted using a predefined key.

When I use the term 'encrypted' I am refering to a process where the data and the key are combined together to create a message which is hopefully unreadable to an attacker. In Osiris, this is done via XORing together the two pieces of data. I'll go into what XOR is in a moment but will mention once more than any attacks detailed against this meter rely on the assumption that a script is injected into the meter. This is a nontrivial task which has not yet been accomplished in the real world, so any attacks are theoretical and rely on conquering that obstacle.

XOR is a bitwise operator which in cryptography combines two values together in a reversible way. It is denoted by the symbol '⊕' but is often denoted as '^' also. A plaintext XORed with a key produces a ciphertext (or encrypted piece of text) and a ciphertext XORed with a key produces the plaintext again. Once the values have been XORed, then new value does not reveal any information about the original two.

It's worth noting that XOR has a bunch of mathematical properties but the most important is that it is commutative, which means that if you're XORing a bunch of things together, you can mess with the order of operations in all manner of ways without impacting the end product.

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

Now I'm not going to illustrate it here, you can do the working on your own, but C1 ⊕ C2 is 11111110. Interestingly, P1 ⊕ P2 is also 11111110. What we've identified here is that given the same key, C1 ⊕ C2 == P1 ⊕ P2. This is another important property of XOR operations. Given two ciphertexts, we can *entirely exclude the encryption from the equation* and deal purely with plaintexts.

If we look back at the previous post about the initalisation of the meter, we'll remember that each script shares knowledge of a predefined password which the security script uses in its first communication. This first communication transmits the 24 character key which the scripts use to encrypt their responses.

For the attacker with a script operating within the context of the meter, it is important to note that we have access to at least 19 ciphertexts encrypted using the same key. Even though this meter is open source, I'll work through this attack as if we know nothing about the internal communications.

Starting from being able to intercept encrypted communications, here is what we see:

    CRheGhVIVVNSQQJdV0gcVkADRRRCUwdeS0UKVRFIVERWQAZdV0hOGgF8FmpZK10RPUcHDGNLFx0pEQ4GAxVwXih/QRk=
    CBBdAUQcGFlVRgVaUEsCVl4DRRRCVwNTQkAGUBZBVFpWXgZdV0gCVgxPBXY0AgQXQgUcM05XAFtSE1dYLgtXH0deBVInWg==
    FxRfA0QEXV9XQAFdUEgCVl4DRRRCVwNWTUEHVhFOVFpWXgZdV0gCVgxPDGcZUmA9MwlZBXgMNxAEOFkfCj5hVDgBPkARWg==
    HhBHB00XBQ4DAkpUU08LVkMLRRRcVwNXSkEDUBVNVlpeSAZdSUgCVkADRVgOVlwsPBRgUFALIw4uNVsmNRYdAUJgJmcdPWsURw==
    HhxULkAWAAYDAkpcUE0FX0cFRRRcVwNXSkEDXxNBUF1QRgZdV1YCVkADRRQOG0IFEhlDK09XKB0KO1suBDZXJTZrTWkjDmIoI0w=
    
The first thing I see here is a series of similar segments which suggests a relationship between the encrypted messages. To be secure, these strings should not be mathematically related to one another, let alone visibly similar. See how at 23 characters in there's a segment of \"DRRRCVw\" with maybe a character's difference on either side? As an attacker, I don't know what this is but it suggests that a 'many time pad'attack exploiting reuse of key may be worthwhile.

The first thing I'm going to do is to XOR each of the ciphertexts with every other ciphertext to make large sample of pairings like C1 ⊕ C2, C1 ⊕ C3, C1 ⊕ C4, C2 ⊕ C3... etc. Because I did this in LSL instead of in a real language (this is not a requirement the attacker has, I just chose to do it this way for kicks) I cut this sample off at 100 pairings, so not all permutations are captured, it's a bit random. Of course, given the information above we know that this is the same as a collection of all XORed plaintext pairs. The attack I'm going to use is best demonstrated using just a single pair, but I'll be applying it to all 100.

If I have C1 ⊕ C2, which is equal to P1 ⊕ P2 and I can guess either P1 or P2 (we'll say I guess P1) I can XOR this pair with P1 as in the following operation:

    P1 ⊕ P2 ⊕ P1

We'd be left with P2, which is a decrypted value the meter is trying to hide from us. Fortunately, because I'm discarding any knowledge given by open sourcing the meter, we don't know P1... but we can probably guess part of it and, if we recall from earlier, the rules that apply to XORing strings also apply to XORing parts of strings.

We're going to assume that somewhere in the communications exists the word \"password\". It's a nice common security word and maybe the meter refers to it. We can do this automatically with a long list of likely words.

Readers will know what the attacker doesn't, that there is a message in the meter which has the plaintext value \"password\|random\_number\|password\_key\". If the attacker cycles through every possible combination of P1 ⊕ P2 and XORs it with the string \"password\" repeated as many times as necessary, any combinations in which \"password\|random\_number\|password\_key\" was either P1 or P2 will reveal part of the other plaintext, specifically the parts that line up with the word \"password\". If \"password\" is not conventiently at the start of the string as it is here, we offset it to test for placement in different locations, trying the following:

    passwordpasswordpasswordpassword
    asswordpasswordpasswordpasswordp
    sswordpasswordpasswordpasswordpa
    swordpasswordpasswordpasswordpas

And so on.

When we XOR a string starting \"password...\" with the XOR'd pairs, we begin to see the start of many of these pairs revealed.

    [05:31] RPCS Original: dmgHandler|fjugohadelbdataln30tYwm;\_$a<w'|#!\"ix fvpj?.4c +?e?rD????#?Y$S??a
    [05:31] RPCS Original: {aps(o.?48w>a8z2ajv*0%pw433!il7	**m+a\?\_?u qaE?4PJ<O<9SbAC8U9K?Bq
    [05:31] RPCS Original: mmzGq969747bipynjederflcvbnm1;{p{j;W$?Q?Tfe`b\L TD(M03Hc??:w.FA(z
    [05:31] RPCS Original: zips%bbdctxaataloaderpic~`hnad{l|data?PM]?a@8dRfe?eYF2\^h\<QmZh?
    [05:31] RPCS Original: main|831747`bqmmedelxlcvbnl17hYvk>\_$?Q?\_gcce\L JZ(M03Hb??)^#GD z
    
Some of these are nonsense, where we XOR \"password\" against a value which has no relationship to it, but anywhere that \"password\" is part of the pair, the other value is revealed.

As an attacker, it's time to take stock of our resources. We have C1 and C2, our ciphertext values. We have part P1 and part of P2. We... hang on. We have Ciphertext and Plaintext partial strings. At the top of this page, I said that Plaintext ⊕ Ciphertext = Key, so can we use this information to recover part of the secret key the meter is using? It turns out yes and the following piece of code will automate this process if dropped into a link\_message event in the meter.

        if (num==8001){
             if(llGetListLength(ciphertexts) < 19) {
                 ciphertexts += str;
                }
        }
        if( num == 8050){
            integer i;
            integer i2;
            for(i = 0;i < 19;i++){
                if(crypto\_key == \"1\"){
                    for(i2 = 0;i2 < 5;i2++){
                        string c1 = llList2String(ciphertexts, i);
                        string c2 = llList2String(ciphertexts, i2);
                        string xortext = llXorBase64StringsCorrect(c1,c2); //This is the same as m1 XOR m2
                        
                        string attempt = llBase64ToString(llXorBase64StringsCorrect(xortext, llStringToBase64(\"dataloader|\")));
                        if(llGetSubString(attempt,0,7) == \"password\"){
    
                            crypto\_key = llGetSubString(llBase64ToString(llXorBase64StringsCorrect(c1, llStringToBase64(\"password\") + randomPass(16))), 0,7) + randomPass(16);
                            llOwnerSay(\"Key is \" + crypto\_key);
                            i=0;
                            i2=0;
                        }
                    }
                } 
                else {
                    llOwnerSay(llBase64ToString(llXorBase64StringsCorrect(llList2String(ciphertexts, i),llStringToBase64(crypto\_key))));
                }
            }
        }
        
This script will gather all ciphertexts as scripts respond to security, then when they are done it will XOR a large number of combinations and use the attacker's knowledge from trying \"password\" to attack the encryption using the string \"dataloader\" which is one of the values that was revealed. Once it has two pieces of plaintext, it uses these in combination with C1 to reveal part of the key. This key partial then had random data added to it to make it as long as a real key, which will allow us to decrypt eight out of every 24 characters as shown in the following decryption output where the real key was dh836grry$y!dkx9i^cn2kia:

    [06:58] RPCS Original: Loading Region Config:Test
    [06:58] RPCS Original: Key is dh836grrttm!k$bdqyq#&6rv
    [06:58] RPCS Original: anticampqi#6>w,l(?\"c$m+'00193323:`$0!*m(?\"1h?P/+Fly2g2Ig?aP\_?l?qo ?}5Y`=
    [06:58] RPCS Original: targetinj,'69{,j-?\"}:m+'00028449;a$0?a*m(?\"}h!]f4KBr/BA4E?#r;~Id\VG.WljeE=
    [06:58] RPCS Original: meter|22;i&8?*s(?\"}$m*/9506300.=`$0?f!|m@;[?Ic+GK14qEIndbaD{\"?A`qp
    [06:58] RPCS Original: comms|51;b-4=*s(?\"}$m, 42894000#`$0?*!diH##?UpO0z2CU1nyeA5U>`vtEE.)
    [06:58] RPCS Original: skills|7>`'2;x*m6?\"}$m+\"58459500#`$0?*!dHq.z?VxtTV78yx]>nSF	.?LE+\")
    [06:58] RPCS Original: ranged|3?i&3:v*m(	\"}$m+'45459910=`:0?*m([n=F?~ p8t/Uo/d<dwa:?i8a?=b?&
    [06:58] RPCS Original: melee|789f%89*m6?\"}$m+&79903000=~$0?*md[k?hHMIxjcYtSzo?{rb	IoP?Y)w`
    [06:58] RPCS Original: GM|226578c$0?a*m(?\"}'j.%304000.0=`$0?3fk{?Y+l?^YvMND4XoGeirsz L?Bb/
    [06:58] RPCS Original: capture|<c-46,m(?<}$m+'09171752=`$.?*m(?n1VkzdJOaeX/Epa?&tY8M?,J?M?V*
    [06:58] RPCS Original: ram|9424:d\"0?4m(?\"}$d/#7289000.=`$0?f!*lT?p?y@TgV55oTi|\_3F?Yh,QSp
    [06:58] RPCS Original: dmgHandlh\"h3:{*i*?\"}$s+'000090975i$0?4m(?\"}$!gfbhhpMn/Lz<\_mL,T8[aJuY?rFOY=
    [06:58] RPCS Original: main|1395b$2?4m(?\"}$j*.900500.0=`$0??Y?P?b?c?SfND0WasSMK?qVi?T?%
    [06:58] RPCS Original: sim|5571<`'0?4m(?\"}$i*&8031000.=`$0?f!ihq???uaG64jB3swB1,kk\"Xe@k&p
    [06:58] RPCS Original: particle~, 4={/j+? c$m+'00386415;a\".?*m(?n1#	uylqVbsNdgf	vQNyX?Ih`fN?#*
    [06:58] RPCS Original: API|1442:e\"0?4m(?\"}$h)\"6095000.=`$0?f!,Nw: kLNYgmLtlbmn\"cDU;\"1vfKp
    [06:58] RPCS Original: passwordqe-8;*i(?\"c$m+'005324918`$0!*m(?\"1h?qDoXPtwhnij>,Jk#J8hIV?Q?+t=
    [06:58] RPCS Original: characte,&6>}-o/?<}$m+'01057024=`$.?*m(?n1cdkq+ZL1iGkZF'RNY}l?alS?|?h*
    
From here, we can start using statistical knowledge of common characters as well as information about how text is encoded when it goes through this process. There are a number of patterns we could use to attack the rest of the string but it's 4am so I think I'm done for now, having hopefully demonstrated that even with a number of layers of at least obfuscation if not secure encryption, an attacker with a script inside your shit is going to mess it up. More secure cryptography has a large overhead as far as memory is concerned and will lag your things to hell and even then there are further cryptographic attacks which can be used to tamper with data even where crypto prevents you from reading it.