---
layout: post
title:  "Attacking Osiris: Introducing Osiris Initialisation"
date:   2015-03-28 00:20:00
categories: security
---

Many years ago in the glory days of Erie Isle, the roleplaying sim I look back on most fondly, I gave some assistance to Colleen Marjeta in securing what was then the Roleplay Combat System (RPCS) and what would eventually be branded Osiris. Having had some measure of success with attacks on DCS (a drama which is another story. TODO: Link to that story here later) it was interesting to apply those lessons to this new meter and actively work against new defences as they were implemented.

I'm going to, as I write this blog series, get the Osiris meter running again from the open source repo and start investigating attacks against it that I had insufficient knowledge for back then.

The primary threat to a combat/RP meter, if it follows the popular paradigm of splitting concerns between multiple scripts and communicating via linked messages, is that an attacker manages to inject their own script into your meter. This gives unrestricted read access to internal communications and allows an attacker to publish their own messages to this forum. As far as LSL is concerned, any messages published within an object are interpreted the same regardless of the script they came from. If an attacker can move your scripts into an object that they control, those communications are exposed.

A popular mitigation to linked message hijacking is to include in the system's integrity checks a function which asks "Was this object created by the system developer?" and/or "Are the scripts inside this object created by the system developer". I've seen and bypassed this security measure in a large number of systems. The reason that this measure can be bypassed is that if a meter is developed by Joe Avatar and only scripts by Joe Avatar can be executed within it, scripts by Joe Avatar become the key to blowing the whole thing open. If Joe Avatar has a long and storied history within SecondLife, there is a reasonable chance that somewhere along the way he released a full perms script and object with no reason to consider that this might endanger his future projects. If Joe Avatar has never released these resources, I have had some luck socially engineering people in this position into releasing full perms content to me. (Read: They gave or sold me a simple notecard dispenser or similar).

After demonstration of this attack, Colleen Marjeta recreated her meter as an alternate account who had never  shared content with anyone. If all of the scripts in your object will only run in an object created by Alice Avatar and Alice never releases an object which can be modified, this attack vector is neutralised.

For the purposes of this research, I'm going to assume that a script has been covertly injected into the object. I'm not looking to work around the tests for object integrity right now, I'm isolating and exploring the key exchange and handshake portion of the meter's intialisation. We'll pretend devious social engineering was responsible for evading the rest.

First of all, for those playing along at home, we're going to need to get the Osiris Combat System running. I have a fork of this repo available at https://github.com/ByteString/osiris-cs/ which at the time of writing this sentence is not suitable for execution but will be updated as soon as I have set up the meter with all of the pre-populated passwords it needs to get running.

Here is a rough description of the initialisation of the Osiris CS.

The main script requests permissions. It then initialises the variable ‘securePass’ which is the prepopulated secret used in the initial key-sharing handshake. Every script knows this secret and it is used during initialisation to encrypt the messages which share with each script which keys they should use going forward. Other scripts may refer to this as ‘initialpass' which is what we’ll call it from now on. The main script is simply reusing the ’securePass’ variable and overwriting this initialpass when the final securePass is agreed upon. 

The main script sends a linked message indicating that the meter’s current state is 99 which means we are undergoing security checks.

checkSecurity() is called which ensures the owner does not have modify permissions and resets the security script, kicking the integrity checking process into action within that script. A timer is set so the main script will from now on, once every second, check whether the security script has come back to confirm that the meter is secure.

If any of these checks fail too hard, the meter nukes itself, wiping its contents.

The security script has woken up now, cleared its history just for good measure and generates a 24 character password using the following function.

    string randomPass(integer length) {
    string letters = "abcdefghijkmnopqrstuvwxyz234567890!@#$$%^&";
    string rPass;
    while(llStringLength(rPass) < length) {
    integer rand = llFloor(llFrand(llStringLength(letters)));
    rPass += llGetSubString(letters,rand,rand);
    }
    return rPass;
    }

The security of this function will be investigated in a future post. The password generated here will be communicated to the scripts after being passed into initialCrypt() with the initialPass. We’ll call this 24 character password ‘securePass' because this is what it is called in most locations in the meter.

Further integrity checks are carried out. The prims in the object are counted. The owner of all of the scripts is confirmed as the meter creator, then the number of scripts is confirmed also, to make sure nothing has snuck in.

The meter now uses initialCrypt() to send a secure challenge containing the 24 character securePass encrypted using the initialpass which all scripts know. This security challenge is a string in the form “security\|securePass\|\|security\_key”.

Now, security\_key is a value we haven’t encountered yet. Every script contains a predefined ‘secureKey' it uses to identify itself and we will refer to these within this post as \<script name>_key. The security script contains a master list of these keys which it uses to confirm that a script is what it says it is and each script knows security_key so they can be sure the security script is talking to them.

A timer is set to trigger every second will check to see whether all scripts have reported back, confirming receipt of the securePass and also identifying themselves as legitimate components of the meter.

Each script receives the challenge from the security script. They decrypt it using their predefined knowledge of the initialpass and they ensure that the first value is the string “security”, the last value is security_key which they have a predefined knowledge of. This is considered sufficient to conclude that this message came from the security script, so the securePass that has been transmitted is stored. Each script then uses this securePass to encrypt a response in the format “\<script name>\|\<random string>\|\|\<secure\_key>” where secure_key is the unique key that script uses to identify itself.

The security script collects all of these responses, ensuring that the scripts correctly identify themselves and prove themselves as legitimate peers to communicate with. It continues to check once a second whether it has received all responses and when it finally has it broadcasts “SECURITYOK\|\|\<the time>”. If it loops more than ten times the meter will destroy itself.

When the main script receives the SECURITYOK signal and confirms that it comes from security, it challenges security to ensure that the security script is legitimate. The main script sends the challenge “MAIN\|\<random number>\|\|main_key” encrypted using the securePass which was distributed earlier.

The security script confirms that main is communicating with it before responding “SECURITY\|\<a new random password>\|\|security_key”. The new randomly generated password is a 12 character password generated using the same function as before. It’s unclear whether this password is used. On receiving this, main loads the character.

And now the meter is running. If you'll excuse me, I'll duck away and get the meter actually running, push that to GitHub and then come back to start talking about how I might attack it.