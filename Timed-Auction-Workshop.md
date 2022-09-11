# Workshop: Timed NFT Auction

In this workshop, we'll go through my Bounty Hack submission, a Timed NFT Auction; where an auctioner can list an nft for the auction and within the allocated time limit bidders are allowed to make their bids, once the time limit is up the highest bidder wins the auction and gets the nft transfered to their address.

This workshop assumes that you've completed the nft auction tutorial.

We assume that you’ll go through this workshop in a directory named ~/reach/workshop-timed-auction:

```$ mkdir -p ~/reach/workshop-timed-auction```

And that you have a copy of Reach installed in ~/reach so you can write

```$ ./reach version```

You should start by initializing your project this would create 2 files in your directory `index.mjs` and `index.rsh`

```$ ./reach init```

## Problem Analysis

I implemented an nft auction using reach but with totally diffrent approach and technologies to ensure my Dapp is fast, effecient and transparent.

A few questions encountered while trying to build this application are:

```
Who is involved in this application?

What information do they know at the start of the program?

What information are they going to discover and use in the program?

What are the steps to be taken in the course of the program?

What funds change ownership during the application and how?
```

**Question Answers!** 
```
Our application involves 2 roles: One deployer(auctioner) and the attachers (numerous bidders).

The auctioner sets the minimum price of the bid, imports the nft and then starts the auction

The attachers(bidders) will be informed of the minimum bid before they make their bids and once any bidder makes a bid the auctioner and the fellow bidders will be informed of the newest bid. 

Bidders can make bids as long as the time isn't up, once the time is up the highest bidder wins and both the auctioner and all the bidders get to see the winners address and his bid.

The Nft is been paid into the contract at the begining of the auction i.e when the contract has been deployed and the highest bidder's bid is been paid into the contract as well and it is then transfered to the auctioner and the nft is transfered to the highest bidder as soon as the time is up.
```

## Data Definition
For the next step, we are going to define the data type equivalents of the values used in our answers from the previous section. Also, in this step we'll be deciding what functions our participants will have.

* What functions/values does the Deployer need to set the parameters of the auction, start the auction and observe the auction till the end ?

* What functions/values does attacher need to enter the auction and make their bids till the auction ends?

* What functions do both participants have in common?

It's time to see our answers!

In this project I used both a prticipant and a particiant class.

First we'll define a common object that contains the functions both participant and participant class have in common.

```js
const Common = {
    seeBid: Fun([Address, UInt], Null),
    showOutcome: Fun([Address, UInt], Null),
}
```
Then we'll define each individual participant's unique function, but also pass the generic ones to both.
```js
export const main = Reach.App(() => {
    setOptions({ untrustworthyMaps: true });

    const Creator = Participant('Creator', {
        ...Common,
        getSale: Fun([], Object({
            nftId: Token,
            minBid: UInt,
            deadline: UInt,
        })),
        auctionReady: Fun([], Null),
        getAddress: Fun([UInt], Address),
    });
    const BidderView = ParticipantClass('BidderView', {
        ...Common,
        optIn: Fun([Token], Null),
    });
        const Bidder = API('Bidder', {
        bid: Fun([UInt], Bool),
    });
    init();

```

On the auctioner side we're going to represent the parameters of the auction in a the function `getSale` and we created an object  which is like a list which is used to store multiple  data type. The Nft Id is represented using a token data type, the minimum bid is represented using UInt(Unsigned Integer), same goes for the deadline.

The function `auctionReady` is used to signify to everyone in the contract that the auction has commenced.

The function `getAddress` is used to get the addresses of the bidders from the frontend array that stores the addresses of each bidder, thats why it has the address data type.

On the bidders side a particpant class was used instead of a regular participant because a participant class can have multiple instances throughout the course of the dapp i.e there can be multiple bidders. This particpant class is identified with `BidderView`, the common object is passed into it and the optin function for the players to opt in to the nft.

For the bidders too we created an API `Bidders` which they can use to make their bids.

## Communication Construction
Now we can design the structure and flow of communication of our application.

```
1. Auctioner sets the minimum bid, deeadline, then pays the nft into the contract then starts the auction.
2. The bidders connect to the contract opt in to the nft and sees the minimum price.
3. The bidders then interact with the API and makes his bid, while they make their bids their bids are been stored in a map and an array as well, once the time is up bidders would not be able to make any more bids.
4. Now for the program to check for the highest bidder a while loop is incoporated, the while loop iterates through the number of bidders.

5. In each loop: 
  i.  The auctioner interacts with the getAddress function passing the counter of the loop to the frontend which is used to index the array that saves the addresses of the bidders then returns the address of the bidder whos address index is the present counter and then stores that address in a variable address
  ii. A function getBid is used to get the indivdual bid of the bidder which was stored in a map earlier by using the address to map his bid. The output of this function is stored in a variable too
  iii. An if statement is used to check if the bidders bid is equal to the maximum integer in the array using array.max()
  iv. If the bid is equal to the the maximum integer the contract transfers the nft to the bidders address and informs everyone in the contract who the winner is and displays the winners bid, else the counter continues and the next loop begins with a new set of address and bid to check.
6. After the loop the contract is emptied and the program ends.
```

The phrase "In each loop" indicates a loop in the auction which  runs according to the number of bidders, until the highest bidder is found and paid. With this information we can implement the logic for our contract.
Main logic of our contract should now look like:

```js
 Creator.only(() => {
        const {nftId, minBid, deadline} = declassify(interact.getSale());
    });
    Creator.publish(nftId, minBid, deadline);

    const amt = 1;
    const x = new Map(Address, UInt);
    commit();
    Creator.pay([[amt, nftId]]);
    Creator.interact.auctionReady();
    BidderView.interact.optIn(nftId);
    BidderView.interact.seeBid(Creator, minBid);

    assert(balance(nftId) == amt, "balance of NFT is wrong");
    const end = lastConsensusTime() + deadline;
    const [
        bidders,
        lastPrice,
        isFirstBid,
        a
    ] = parallelReduce([0, minBid * 1000000, true, array(UInt, [0, 0, 0, 0, 0])])
        .invariant(balance(nftId) == balance(nftId))
        .while(lastConsensusTime() <= end && bidders < 5 )
        .api_(Bidder.bid, (bid) => {
            
            check(bid * 1000000 > lastPrice, "bid is too low");
            return [ bid * 1000000, (notify) => {
                x[this] = bid
                notify(true);
                Creator.interact.seeBid(this, bid);
                BidderView.interact.seeBid(this, bid);
                return [bidders+1, bid * 1000000, false, a.set(bidders, bid)];
            }];
        })
        .timeout(absoluteTime(end), () => {
          Creator.publish();
          return [bidders, lastPrice, isFirstBid, a];
      });
    commit();
    Creator.publish()

    var [index, winner ] = [ 0, Creator ]
    invariant(balance(nftId) == balance(nftId))
    while( index < bidders ){
        commit()
        Creator.only(() => {
            const address = declassify(interact.getAddress(index))
        })
        Creator.publish(address);
        
        const b = getBid(address, x);

        if (b == a.max()){
            transfer(balance(nftId), nftId).to(address);
            [ index, winner ] = [ index + 1, address ];
            continue;
        }
        else{
            [ index, winner ] = [ index + 1, winner ];
            continue;
        }
    }
        transfer(balance()).to(Creator);
        transfer(balance(nftId), nftId).to(Creator);
        Creator.interact.showOutcome(winner, lastPrice);
        BidderView.interact.showOutcome(winner, lastPrice);
    commit();
    exit();
});
```
## Assertion Insertion
Due to simplicity of the program, there's no need for assertions in the code.

## Possible Additions
My code works perfectly fine as it is now. But the the range of bidders can be increased but increasing the inital values of the content of the array used to store the bidders bid in the api. This is because the size of arrays in reach are a lot more controlled compared to our traditional arrays in javascript and this is like this to increase the security and the credibility of our smart contract.

## Testing 
We test our application by creating a file `index.mjs` in the same directory as the `index.rsh`
```bash
touch index.mjs
```
We define our test data to use for simulating user input and data
```js
import { loadStdlib } from '@reach-sh/stdlib';
import * as backend from './build/index.main.mjs';

const stdlib = loadStdlib();
const startingBalance = stdlib.parseCurrency(100);

console.log(`Creating test account for Creator`);
const accCreator = await stdlib.newTestAccount(startingBalance);

console.log(`Having creator create testing NFT`);
const theNFT = await stdlib.launchToken(accCreator, "bumple", "NFT", { supply: 1 });
const nftId = theNFT.id;
const minBid = stdlib.parseCurrency(2);
const deadline = 50;
const params = { nftId, minBid, deadline };

let done = false;
const add = [];
const bidders = [];
const startBidders = async () => {
    let bid = minBid;
    const runBidder = async (who) => {
        const inc = stdlib.parseCurrency(Math.random() * 10);
        bid = bid.add(inc);
        const acc = await stdlib.newTestAccount(startingBalance);
        acc.setDebugLabel(who);
        await acc.tokenAccept(nftId);
        const addr = acc.getAddress()
        bidders.push([who, acc]);
        add.push(addr);
        const ctc = acc.contract(backend, ctcCreator.getInfo());
        const getBal = async () => stdlib.formatCurrency(await stdlib.balanceOf(acc));

        console.log(`${who} decides to bid ${stdlib.formatCurrency(bid)}.`);
        console.log(`${who} balance before is ${await getBal()}`);
        try {
            const [ lastBidder, lastBid ] = await ctc.apis.Bidder.bid(bid);
            console.log(`${who} out bid ${lastBidder} who bid ${stdlib.formatCurrency(lastBid)}.`);
        } catch (e) {
            console.log(`${who} failed to bid, because the auction is over`);
        }
        console.log(`${who} balance after is ${await getBal()}`);
    };


    await runBidder('Alice');
    await runBidder('Bob');
    await runBidder('Claire');
    await runBidder('Etuk');
    while ( ! done ) {
        await stdlib.wait(1);
    }
};

const ctcCreator = accCreator.contract(backend);

await ctcCreator.participants.Creator({
    getSale: () => {
        console.log(`Creator sets parameters of sale:`, params);
        return params;
    },
    auctionReady: () => {
        startBidders();
    },
    seeBid: (who, amt) => {
        console.log(`Creator saw that ${stdlib.formatAddress(who)} bid ${stdlib.formatCurrency(amt)}.`);
    },
    showOutcome: (winner, amt) => {
        console.log(`Creator saw that ${stdlib.formatAddress(winner)} won with ${stdlib.formatCurrency(amt)}`);
    },
    get_Address: async (count) => {
        return add[count]
    },
    get_bid: async (t) => {
        return [parseInt(stdlib.bigNumberToNumber(t[1]))]
    },

    /*timesup: () => {
        console.log('I think time is up');
        ctcCreator.apis.Bidders.timesUp();
      }*/
});


for ( const [who, acc] of bidders ) {
    const [ amt, amtNFT ] = await stdlib.balancesOf(acc, [null, nftId]);
    console.log(`${who} has ${stdlib.formatCurrency(amt)} ${stdlib.standardUnit} and ${amtNFT} of the NFT`);
}
done = true;
```
## Further Learning
For a link to the repo [CLICK HERE](https://github.com/Utibetuk/Timed-Auction).

I used react js, html and css to create the UI of the Dapp, if you are intrested in building up yours too you can check out my repo above but I would leave the full `index.rsh` and `App.js` code below for your further learning purposes.

### Full `index.rsh` code

```js
"reach 0.1";

const getBid = (address, bidMap) => {
  const bid = bidMap[address].match({ 
    Some: (number) => number,
    None: () => 0,
  });
  return bid;
}

const Common = {
    seeBid: Fun([Address, UInt], Null),
    showOutcome: Fun([Address, UInt], Null),
}
export const main = Reach.App(() => {
    setOptions({ untrustworthyMaps: true });

    const Creator = Participant('Creator', {
        ...Common,
        getSale: Fun([], Object({
            nftId: Token,
            minBid: UInt,
            deadline: UInt,
        })),
        auctionReady: Fun([], Null),
        getAddress: Fun([UInt], Address),
    });
    const BidderView = ParticipantClass('BidderView', {
        ...Common,
        optIn: Fun([Token], Null),
    });
    const Bidder = API('Bidder', {
        bid: Fun([UInt], Bool),
    });
    init();

    Creator.only(() => {
        const {nftId, minBid, deadline} = declassify(interact.getSale());
    });
    Creator.publish(nftId, minBid, deadline);

    const amt = 1;
    const x = new Map(Address, UInt);
    commit();
    Creator.pay([[amt, nftId]]);
    Creator.interact.auctionReady();
    BidderView.interact.optIn(nftId);
    BidderView.interact.seeBid(Creator, minBid);

    assert(balance(nftId) == amt, "balance of NFT is wrong");
    const end = lastConsensusTime() + deadline;
    const [
        bidders,
        lastPrice,
        isFirstBid,
        a
    ] = parallelReduce([0, minBid * 1000000, true, array(UInt, [0, 0, 0, 0, 0])])
        .invariant(balance(nftId) == balance(nftId))
        .while(lastConsensusTime() <= end && bidders < 5 )
        .api_(Bidder.bid, (bid) => {
            
            check(bid * 1000000 > lastPrice, "bid is too low");
            return [ bid * 1000000, (notify) => {
                x[this] = bid
                notify(true);
                Creator.interact.seeBid(this, bid);
                BidderView.interact.seeBid(this, bid);
                return [bidders+1, bid * 1000000, false, a.set(bidders, bid)];
            }];
        })
        .timeout(absoluteTime(end), () => {
          Creator.publish();
          return [bidders, lastPrice, isFirstBid, a];
      });
    commit();
    Creator.publish()

    var [index, winner ] = [ 0, Creator ]
    invariant(balance(nftId) == balance(nftId))
    while( index < bidders ){
        commit()
        Creator.only(() => {
            const address = declassify(interact.getAddress(index))
        })
        Creator.publish(address);
        
        const b = getBid(address, x);

        if (b == a.max()){
            transfer(balance(nftId), nftId).to(address);
            [ index, winner ] = [ index + 1, address ];
            continue;
        }
        else{
            [ index, winner ] = [ index + 1, winner ];
            continue;
        }
    }
        transfer(balance()).to(Creator);
        transfer(balance(nftId), nftId).to(Creator);
        Creator.interact.showOutcome(winner, lastPrice);
        BidderView.interact.showOutcome(winner, lastPrice);
    commit();
    exit();
});
```

### Full `app.js` code
```js
import './App.css';
import { loadStdlib } from '@reach-sh/stdlib';
import { ALGO_MyAlgoConnect as MyAlgoConnect } from '@reach-sh/stdlib';
import * as backend from './reach/build/index.main.mjs'
import { useEffect, useState } from 'react';
import { views, Loader } from './utils/';
import { ConnectAccount, PasteContractInfo, SelectRole, ViewAuction, ViewWinner, WaitForAttacher } from './screens';


const reach = loadStdlib('ALGO');
reach.setWalletFallback(reach.walletFallback( { providerEnv: 'TestNet', MyAlgoConnect } ));
// const fmt = (x) => reach.formatCurrency(x, 4);

function App() {
  const [ account, setAccount ] = useState({})
  const [ view, setView ] = useState(views.CONNECT_ACCOUNT)
  const [ contractInfo, setContractInfo ] = useState(`{"type":"BigNumber","hex":"0x0"}`)
  const [ contract, setContract ] = useState()
  const [ resolver, setResolver ] = useState({
    resolve: ()=>{}
  })
  const [ trigger, setTrigger ] = useState(false)
  const [ index, setIndex ] = useState(0)
  const [ bid, setBid ] = useState([
    // { address: '0x0blahblah', bid: 2 },
    // { address: '0x0blahblah', bid: 2 },
  ])
  const [ winner, setWinner ] = useState({ address: '0x0blahblah', bid: 2 })

  useEffect(() => {
    if(trigger === true){
      console.log(bid[index].address)
      resolver.resolve(bid[index].address)
      setTrigger(false)
    }
  }, [bid, index, resolver, trigger])

  const reachFunctions = {
    connect: async (secret, mnemonic = false) => {
      let result = ""
      try {
        const account = mnemonic ? await reach.newAccountFromMnemonic(secret) : await reach.getDefaultAccount();
        setAccount(account);
        setView(views.DEPLOY_OR_ATTACH);
        result = 'success';
      } catch (error) {
        result = 'failed';
      }
      return result;
    },

    setAsDeployer: (deployer = true) => {
      if(deployer){
        setView(views.SET_TOKEN_INFO);
      }
      else{
        setView(views.PASTE_CONTRACT_INFO);
      }
    },

    deploy: async (name, url, minBid) => {
      setView(views.DEPLOYING);
      const contract = account.contract(backend);
      const NFT = await reach.launchToken(account, name, 'NFT', { supply: 1, url })
      // setNftId(NFT.id)
      backend.Creator(contract, {
        ...Creator,
        getSale: () => {
          return {
            nftId: NFT.id,
            minBid,
            deadline: 50
          }
        }
      });
      const ctcInfo = JSON.stringify(await contract.getInfo(), null, 2)
      setContractInfo(ctcInfo);
      setView(views.WAIT_FOR_ATTACHER)
    },

    attach: (contractInfo) => {
      const contract = account.contract(backend, JSON.parse(contractInfo));
      setContract(contract)
      backend.BidderView(contract, BidderView)
      setView(views.ATTACHING)
    }
  }

  const Common = {
    seeBid: ( address, bid) => {
      setBid(b => {
        const copy = [...b]
        copy.push({
          address,
          bid: parseFloat(bid)
        })
        return copy
      })
    },

    showOutcome: (address, bid) => {
      setWinner({
        address,
        bid: parseFloat(bid)
      })
      setView(views.VIEW_WINNER)
    }
  }

  const Creator = {
    ...Common,

    auctionReady: () => {},

    getAddress: (indexHex) => {
      setIndex(parseInt(indexHex))
      return new Promise(resolve => {
        setResolver({
          resolve: (a) => resolve(a)
        })
        setTrigger(true)
      })
    }
  }

  const BidderView = {
    ...Common,

    optIn: async (tokenID) => {
      setView(views.OPT_IN)
      const id = reach.bigNumberToNumber(tokenID) //accept token
      await account.tokenAccept(id)
      setView(views.VIEW_AUCTION)
    }
  }

  
  return (
    <div className="App">
      <div className='top'>
        <h1>Timed Auction</h1>
      </div>
      <header className="App-header">
        {
          view === views.CONNECT_ACCOUNT && 
          <ConnectAccount connect={reachFunctions.connect}/>
        }

        {
          view === views.DEPLOY_OR_ATTACH &&
          <SelectRole deploy={reachFunctions.deploy} attach={() => setView(views.PASTE_CONTRACT_INFO)}/>
        }

        {
          (view === views.DEPLOYING || view === views.ATTACHING) &&
          <Loader />
        }

        {
          view === views.OPT_IN && 
          <>
            <Loader />
            <h5>Opting Into NFT. . .</h5>
          </>
        }

        {
          view === views.PASTE_CONTRACT_INFO &&
          <PasteContractInfo attach={reachFunctions.attach}/>
        }

        {
          view === views.WAIT_FOR_ATTACHER &&
          <WaitForAttacher info={contractInfo} bid={bid}/>
        }

        {
          view === views.VIEW_AUCTION &&
          <ViewAuction 
            bid={bid} 
            placeBid={async (amount) => {
              try {
                await contract.apis.Bidder.bid(amount)
                return true
              } catch (error) {
                console.log(error)
                return false
              }
            }}
          />
        }

        {
          view === views.VIEW_WINNER &&
          <ViewWinner winner={winner} reset={() => setView(views.DEPLOY_OR_ATTACH)}/>
        }
      </header>
    </div>
  );
}

export default App;

```
The front-end structure is intentionally easy to understand so you'll have to properly understanding when trying to build yours out.

## Discussion
Congrats on finishing this workshop. You implemented a timed nft auction that runs on the blockchain yourself.

If you found this workshop rewarding please let us know on [the Discord Community](https://discord.gg/AZsgcXu).

Thanks!!

