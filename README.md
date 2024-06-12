# FareEnough ~ The Price of Oversight

-----
## Introduction
In my investigation of NFC (Near Field Communication) Cards utilized within Public Transit Systems, I investigated their operational details, security frameworks, and information processing methods.

This write-up aims to offer an extensive understanding for a broad readership by presenting explanations that encompass both fundamental and intricate aspects.

-----
### Equipment
For this research project, I used the following Hardware/Software/Applications/Cards:
- Proxmark3 Easy (512kb) running the [Iceman Firmware](https://github.com/RfidResearchGroup/proxmark3)
  - > A hardware platform used for interacting with and investigating High and Low-frequency RFID credentials such as the ones used at this Transit Authority.
- Gen2/CUID Magic Mifare Ultralight (EV1-UL11) Cards
  - > An advanced RFID card commonly used for access control and identification, compatible with Mifare Ultralight EV1-UL11 standards.
    - >> This modified card allows for data to be written to memory blocks that are usually `Locked` by the manufacturer (`UID`).
- Gen4 Ultimate Magic Card
  - > A cutting-edge RFID cards designed for high-security access control and identification systems, featuring advanced capabilities and enhanced security measures.
    - >> This modified card allows for data to be written to memory blocks that are usually `Locked` by the manufacturer (`UID` & `Signature`) as well as a newer functionality called `Shadow-Mode`
- Single-Use (Day Pass) issued NFC Card
  - > A Single-Use NFC Day Pass, issued by a Ticket Vending Machine (TVM), grants temporary access to public transit for a day.
- Android Phone with the `MetroDroid` Application
  - > View remaining balance, recent trips, and other info from contactless public transit cards using NFC on Android.
- Android Phone with the `TagInfo` Application by NXP
  - > Provides detailed information about NFC tags and RFID cards, aiding developers and enthusiasts in understanding their capabilities and content.

-----
### The Card in Question
The NFC Card used by this Transit Authority was powered by the [Mifare Ultralight EV1 (48 Byte)](https://www.nxp.com/docs/en/data-sheet/MF0ULX1.pdf), a widely deployed High-Frequency chipset. These chips are prevalent across various industries due to their convenience and are widely used within the Transit Industry. 

A simple scan revealed its identity: 

``` hf search ```

![PM3](https://i.imgur.com/bIuDYlI.png)

```
Result:
- UID: 04 B0 41 CA E9 14 91
- ATQA: 00 44
- SAK: 00 [2]
- Card Type: MIFARE Ultralight EV1
```

The Mifare Ultralight EV1 chipset, while widely used, possesses exploitable vulnerabilities, which were a primary focus of my investigation.

-----
### Double Trouble
To begin my investigation I decided to download the data from a brand new & un-scanned card, so I could get the baseline data.  I wanted to know what data would change, once the card was scanned through the turnstile System. 

I also decided to take the download from above, and write that data to a Gen2 / Mifare Ultralight EV1 Magic Card.  To see if the turnstile tracked cards by UID or by data written on memory blocks. 
> *A Gen2 Magic Card, which is a special type of Mifare Ultralight, used in duplication operations, as it allows all of its data to be changed to reflect that of another Mifare Ultralight including its Unique Identifier.* 

After going through the turnstile with the Original card, causing the data on the card to be altered via the turnstile software.  I applied the duplicated card to the reader, which once again, gave me access through the turnstile.

This was interesting, as this means that the card itself must be locally storing the "Balance" on the card and not utilizing any kind of back-end validation system to track which cards have been used and allowed through.

> While this is the intended purpose of Mifare Ultralight cards, due to the insecurity of having this data stored locally and untracked digitally, most places that would use these cards, instead use a validation system, that would allow the cards to be tracked by the UID or a combination of data stored on memory blocks, in tandem with tracking the UID..
>> IE: Arcades tend to use this type of card solely as an identifier which is then used to look up the user's information on a back-end system where the value is stored and retrieved as necessary.

-----
## Original Card:
  ![Vonfidex](https://i.imgur.com/esuJh6f.png)
  
-----
### Data Transfer and Replication
With the `Proxmark3 Easy` at my disposal, I performed successful scans and data extraction from the Transit Authority Card.

The extracted data formed the foundation for the subsequent duplications of the card's information onto a blank `Gen2/CUID Magic Mifare Ultralight (EV1-UL11)` Card as well as copying the data to a `Gen4 Ultimate Magic Card` , setting the stage for comprehensive testing.

``` hf mfu dump --ns ```

```
Result:
- Successfully dumped the data from the current card
```

![Dump_ns](https://i.imgur.com/Q6auqIK.png)

-----
### Discovering the 'Source of Truth'
An alarming realization surfaced during my early tests at the Transit Station.  I had discovered the `source of truth`, for determining the card's transit balance/expiry period, resided solely within the card itself.

This revelation contrasted sharply with my initial assumption. I assumed data would be centrally administered on a server or within the turnstile itself.

-----
### Local Values
When analyzing the memory of the original **Unscanned** credential with that of the **Post-Scanned** credential, it revealed discrepancies from Blocks 12 to 15 of the Mifare Ultralight.

These differences strongly suggest any stored value resided in these blocks and are written by the turnstile. This leads me to believe once these memory blocks are what house the 'expiry offset' data.

![Diffs](https://i.imgur.com/IfEyl05.png)

-----
#### Original UnScanned Data:
  - Block 12:  `00000000`
  - Block 13:  `00000000`
  - Block 14:  `00000000`
  - Block 15:  `0000C1FB`
#### Original PostScanned Data:
  - Block 12:  `667C5807`
  - Block 13:  `82F10100`
  - Block 14:  `573D0001`
  - Block 15:  `0000181C`
> *When Attempting to convert the 4 bytes in each block to ASCII. It had resulted in Non-Readable Characters*
 
-----
## Editing and Testing NFC Cards
Now I knew that the data was stored locally on the cards and where the information was in the data blocks.

I decided that the next step would be to start editing the card's data for testing purposes. This would allow me to determine just how serious of a vulnerability this could potentially be.

> In the following phase of the attempted exploitation, the objective was to modify the card's data and write it onto a new card for the purpose of testing.
### Data Dump
The card's data was successfully extracted into a ```.json``` file, allowing for specific line modifications. 

The initial data extraction presented as follows:
- "12": "00000000",
- "13": "00000000",
- "14": "00000000",
- "15": "0000C1FB",

-----
### Block 15 Issues
After altering the data between Blocks 12 & 15, I determined that `Block 15` was some form of *CheckSum* put in place, to stop individuals from altering locally stored data.

Block 15:  `0000C1FB`

While normally, this is a good way to stop individuals from altering the data and setting the values to whatever they want.  This method isn't full-proof though, and we'll discuss why in a little bit.

-----
### Writing to the cloned card (Attack A)

Since I knew that the `Baseline` could be Re-Written and Re-Used, within the expiry period of 90 days, I decided to test the limits of this "Attack".

As I was aware cloning the data onto a `Gen2/CUID Magic Mifare Ultralight (EV1-UL11) Card` worked without any issues.

I constructed a test to see how the turnstile would interact with a `Gen4 Ultimate Magic Card`.
> What makes the `Gen4 Ultimate Magic Card` unique, is that it has the ability to run in what is called a *`Shadow Mode`*.  What Shadow Mode allows for, is it's ability to have Baseline Data written to it and then allow data to be temporarily written to it.

What makes this worth investigating further, is the fact that I can write the `Pre-Scanned Data` to the Gen4 Card as it's `Baseline Data`.  Then, I can have the `Expiry Data` that is written to the card, be Temporary.

![Gen2](https://i.imgur.com/5s8Ap5c.png)

-----
### Testing the Edited Card (Attack B)
With the newly cloned and edited card in hand, I went back to the Transit Station to test if my hypothesis was correct.

After scanning through with my Original Card, to verify that the data on the card was still legitimate and working properly.  I then scanned through with the `Gen4 Card`. 

Once I was through the turnstile, I used my Android Phone with the `MetroDroid` Application to see if the Expiry Offset data was stored on the card and I ReVerified with the `TagInfo` Application.
> On the Original Card, the Offsetting data was written and stored.
> On the `Gen4` Card, the Offsetting data was temporarily written to the card and fell off by the time I scanned it with either application.
> > What this allowed me to do, is I can now use the `Gen4` card as many times as I'd like, within a 90 day period.  I will no longer need to Re-Write the data back to the `Baseline Data`.

![Gen4](https://i.imgur.com/ooCQTQf.png)

-----
### Editing the Original Card (Attack C)
I wanted to learn what I would need to do, if I did not have access to a `Gen4` Card and I still wanted to exploit this system.

I learned that one thing I could do, would be to save the `Pre-Scanned` data from the Original Card, and continuously Re-Write that *`Baseline`* data to the Original Card.
> Since I do now know the algorithm used for the cards CheckSum, I am not able to manipulate the data on other cards that have already been used, without knowing what the PreScan data is.
> > This is because it is assumed that the CheckSum in `Block 15` uses data from `Block 0` *(UID)*

What this allowed me to do, would be to:
- Buy a *"Single Use"* Ticket for $5
	- Save the Original Un-Scanned Data from that ticket
		- Use that ticket as normal

Once the day has finished I can:
- Re-Write the `Baseline Data` to the card
	- Repeat previous steps for a total of 90 days

Once the 90 Day "Expiry Period" has ended, I would then need to purchase another $5.00 Ticket and I can continue on with the above.

-----
### What's the Impact of this Vulnerability?
The following are the prices for each type of Pass that available for Purchase:
- 1-Day Pass
	- $5
- 3-Day Pass
	- $15
- 7-Day Pass
	- $20
- 30-Day Pass
	- $75

> For the price of a `1-Day Pass` ($5), I am able to save $220 for 90 Days of use.
>> Over the course of 1 Year, this would cost the Transit Authority approximately $880.  Assuming that an Individual used the Pass every day, for 1 year.

-----
## Incorporating Additional Insights
### Mifare Ultralight EV1 (48 Byte) Card Vulnerabilities
In the world of NFC card security, the implementation of the Mifare Ultralight EV1 chipset is a recurring concern. Its inherent vulnerabilities make it susceptible to exploits, allowing attackers to recover sector keys and access data within the card. Tools such as the Proxmark client make this process relatively straightforward. 

![NXP](https://i.imgur.com/RTE157z.png)

-----
### Local Storage of Card Data
One of the notable revelations during my investigation was that the storage of `Expiry Data`, was stored on the card itself and then the `Offsetting Expiry Data` was then written to the card via the turnstile Reader/Writer.

Unlike systems that rely on centralized servers for Card Usage Validation, the Mifare Ultralight EV1 card stored this information internally and had no other Checks/Balances. While this approach aligns with its intended use, it introduces security risks, especially if the card is lost or stolen.

-----
## Conclusion
By utilizing the technique discussed via `"Attack A"`, I was able to take data from an unused `Single-Use` Transit Card and clone that data to a `Gen2` card.  At the bare minimum this allows me to make duplicated cards, assuming that I have enough blank `Gen2` cards to keep making copies.  Alternatively, I can continue to re-write the original data to the `Gen2` card and continue to exploit this vulnerability that way.

By utilizing the technique discussed via `"Attack B"`, I was again able to take data from an unused 
`Single-Use` Transit Card and clone that data to a `Gen4` card.  With the `Gen4` card, it would save me the time and effort of needing to constantly re-write the data onto the card after each days usage, due to the functionality of `Shadow-Mode`

By utilizing the technique discussed via `"Attack C"`, I am able to still exploit this vulnerability, even without any specialized cards.  I can continuously re-write the data from an unused `Single-Use` Transit Card to the Post-Scanned `Single Use` Ticket that the data came from.  This would allow me to continuously re-use the ticket for 90 days, without the need of any specialized cards.
