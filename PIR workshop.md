<p align="center">
  <a href="" rel="noopener">
 <img src="https://docs.reach.sh/assets/logo.png" alt="Project logo"></a>
</p>
<h3 align="center">The Price Is Right - Casino Edition</h3>

<div align="center">
  

</div>

---

<!-- <p align="center"> Workshop : Price Is Right Casino Edition
    <br> 
</p> -->

 
In this workshop, we'll be creating a casino-like decentralized application in which 2 players place their wagers(bets) and are tasked to guess the right number to win back both their wager and their opponent's.  
  
This workshop is for a project that was created during the Decentralized Umoja3 Hackathon.  
  
We assume that you'll go through this workshop in a directory named `~/reach/price-the-right-casino-edition`
  
Using the below terminal command, we will be creating and accessing a new directory  
```bash
$ mkdir -p ~/reach/price-the-right-casino-edition && cd ~/reach/price-the-right-casino-edition
```
Next, we download reach using the below command   
  
```bash
curl https://docs.reach.sh/reach -o reach ; chmod +x reach
```
Once that is done, to confirm that reach has been installed successfully, we will run the following command 

```bash 
$ ../reach version
```
The output of that command should diplay the latest reach version.  

Next, we initialize our reach application using the following command 
  
```bash
./reach init
```
This creates an `index.mjs` which is the test interface and an `index.rsh` file which is the smart contract, also known as the Reach file.  
  
# Problem Analysis 
  
As a developer, before you start writing any line of code to perform any function, we'll have to ask ourselves some questions and produce our answers, then write down an algorithm of how we want that application to work and how it solves the problems listed then and only then, we can start writing our first lines of code.  
  
For this Dapp we have to ask ourselves the following questions:
  
```
1. What is the logic of the game?
```
```
2. Why is it called Casino Edition?
```
```
3. How are the funds being moved?
```
Please sit tight and enjoy, the questions above will be answered as workshop continues.  
  
# Application Logic  
We would start by answering the very first question which concerns the logic of our application.
The logic goes like this:
```
Step 1 - The House aka Deployer sets the game parameters  which are the deadline, the wager, the range of the random numbers and the number of trials the players are allowed to have then the Deployer deploys the contract.
```
```
Step 2 - The two players Alice and Bob then attach to the contract using the contract info given by the House.
```
```
Step 3 - The two players are prompted to accept the wager and the rules of the game before the game commences.
```
```
Step 4 - Once they have both accepted the wager and the rules of the game they are allowed to start the game with Alice having to guess the random number in a given range of numbers then bob is allowed to do the same.
```
```
Step 5 - The game continues until either of the players have guessed the right number or they have both exceeded the number of trials set by the house.  
```
```
Step 6 - If Alice only guesses the right number before the trials are exceeded she wins the game and gets both her wager and Bob's  wager, if Bob only guesses the right number before the trials are exceeded he wins the game and gets both his  wager and Alice's wager, if they  both guess the right number before the trials are exceeded they get refunded and if none of them get the right guess and their trials have been exceeded they both lose their wagers to the house and are prompted to play again if they want to.  
```
If you have been following up, I belive everything is clear at this point and we have answered our first question.  
  
# Logic Implementation  
  
Now that we have an algorithm of how our application is meant to work and the logic it has to process to give our desired outcome we can start writing our first lines of code.  
  
If you open the directory on your editor most preferably VS CODE we would see 3 files in it which are `index.mjs`, `index.rsh` and `reach`.  
  
The file reach is what we downloaded earlier into the directory. If you open the `index.rsh` and the `index.mjs` files you will see the following lines of code:  
  
`index.rsh`
```js
'reach 0.1';
  
export const main = Reach.App(() => {
  const A = Participant('Alice', {
    // Specify Alice's interact interface here
  });
  const B = Participant('Bob', {
    // Specify Bob's interact interface here
  });
  init();
  // The first one to publish deploys the contract
  A.publish();
  commit();
  // The second one to publish always attaches
  B.publish();
  commit();
  // write your program here
  exit();
});
```
`index.mjs`
```js
import {loadStdlib} from '@reach-sh/stdlib';
import * as backend from './build/index.main.mjs';
const stdlib = loadStdlib(process.env);

const startingBalance = stdlib.parseCurrency(100);

const [ accAlice, accBob ] =
  await stdlib.newTestAccounts(2, startingBalance);
console.log('Hello, Alice and Bob!');

console.log('Launching...');
const ctcAlice = accAlice.contract(backend);
const ctcBob = accBob.contract(backend, ctcAlice.getInfo());

console.log('Starting backends...');
await Promise.all([
  backend.Alice(ctcAlice, {
    ...stdlib.hasRandom,
    // implement Alice's interact object here
  }),
  backend.Bob(ctcBob, {
    ...stdlib.hasRandom,
    // implement Bob's interact object here
  }),
]);

console.log('Goodbye, Alice and Bob!');
```
In the `index.rsh` file  we would start by creating the `winner` function that we would be using in the game to compute the outcome of the game using the guesses of both Alice and Bob and the random number as arguments in the function.
```js
"reach 0.1";

const [ isOutcome, BWINS, DRAW, AWINS ] = makeEnum(3); 

//This computes the winner of the game
const winner = (rand,hand1, hand2) => {
  const x = rand;
  if (hand1 == x && hand2 != x){
    return AWINS;
  }
  else if(hand1 != x  && hand2 == x){
    return BWINS;
  }
  else if (hand1 == x && hand2 == x){
    return DRAW;
  }
  else  return DRAW;

};
```
Next, we would be creating another function(`payWinner`) that is in charge of disbursting the wagers according to the outcome of the game.  
  
```js
// Makes the required payment to the winner
const payWinner = (outcome, wager, Alice, Bob, House) => {
  if (outcome == DRAW) {
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome)
    });
    transfer(2*wager).to(House);
  }
  else if(outcome == A_WINS) {
    transfer(2*wager).to(Alice);
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome);
    });
  }
  else {
    transfer(2*wager).to(House);
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome);
    });
  }
}
```
Next, we define the players abilities:
- `seeRules` is the function the players use to see the rules of the game.
- `getHand` is a function the players use to make their guesses.
- `seeOutcome` is a function the players use to see the outcome of the game after each round.
- `informTimeout` is used to implement a timeout if any player decides to delibrately  delay the game.
- `informNewRound` is used to signify a new round.

```js
const Common  =  {
    ...hasRandom
};
//Player abilities
const Player = {
  seeRules: Fun([UInt, UInt], Null),
  getHand: Fun([UInt], UInt),
  seeOutcome: Fun([UInt], Null),
  informTimeout: Fun([], Null),
  informNewRound: Fun([], Null),

};
```
The snippet below is used to signify House, Alice and Bob interfaces including their individual abilites.  
We initalize the reach application with the `init()`.  
  
The reason why this is a casino edition is because, instead of having just 2 participants in the game, we have 3 in which one is the House aka Organizer and the role of the House is to spectate the game, set the rules and also collect the wagers if none of them win.   
  
```js
export const main = Reach.App(() => {
// House Interface
const House = Participant('House', {
    ...Common,
    wager: UInt,
    limit: UInt,
    trials: UInt,
    deadline: UInt,
    rand: Fun([UInt], UInt),
    waitingForAttacher: Fun([], Null),
    seeOutcome: Fun([UInt], Null),
})
//Alice interface
  const Alice = Participant('Alice', {
    ...hasRandom,
    ...Player,
    acceptWager: Fun([UInt], Null), 
    //waitingForAttacher: Fun([], Null)
  });
//Bob interface
  const Bob   = Participant('Bob', {
    ...hasRandom,
    ...Player,
    acceptWager: Fun([UInt], Null),
  });
  init();

```
- Here the `House` interacts with his functions hereby setting all the parameters of the  game then deploys the contract and waits for the players to attach.  
  
```js
const informTimeout = () => {
    each([Alice, Bob], () => {
      interact.informTimeout();
    });
  };
//Alice and Bob accept the rules and pay the wager
  House.only(() => {
    const limit = declassify(interact.limit);
    const trials = declassify(interact.trials);
    const wager = declassify(interact.wager);
    const deadline = declassify(interact.deadline);
    const rand = declassify(interact.rand(limit));
  });
  House.publish(limit, trials, wager, deadline, rand)
  commit();

  House.interact.waitingForAttacher();
```
- Here Alice accepts the rules of the game by interacting with the seeRules function and depending on her response the game will continue or  end, she is also prompted to accept the wager then she publishes her response to the blockchain and pays the wager into the contract and the same goes for Bob.  
  
```js
 Alice.only(() => {
    interact.seeRules(limit, trials);
    interact.acceptWager(wager);
  });
  Alice.publish();
  commit();
  Alice.pay(wager);
  commit();


  Bob.only(() => {
    interact.seeRules(limit, trials);
    interact.acceptWager(wager);
    
  });
  Bob.publish()
  .pay(wager)
    .timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));
```
Once all the players have accepted the wagers and have paid in the wagers the next step is the start of the game.  
- A while loop is used to implement the feature of the game looping for the amount of trials set by the house and also until either of the players guess the right number.  
- In reach we must use a var to represent mutable values i.e values that can change.  
- A while loop in reach is not complete without its invariant. An invariant is a statement that must be true before , during and after the while loop, it is  necessary for reach programs during the verification process.  
- In this while loop, Alice interacts with her `getHand` fuction inputing her guess then Bob does the same then their guesses are been assigned to a var at the end of the loop, these mutable variables are used in the conditions of the while loop to  know if theres a winner.  
  
```js
//While loop that loops as long as  the conditions are not met
  var [stage, hand1, hand2 ] = [trials,0,0];
  invariant( balance() == 2 * wager);

  while ((hand1!= rand && hand2 != rand) && (stage > 0)) {
    commit();

    Bob.interact.informNewRound();
    Alice.interact.informNewRound();

    Alice.only(() => {
      const _handAlice = interact.getHand(limit);
      
      const [_commitAlice, _saltAlice] = makeCommitment(interact, _handAlice);
      const commitAlice = declassify(_commitAlice);
      });
    Alice.publish(commitAlice)
      .timeout(relativeTime(deadline), () => closeTo(Bob, informTimeout));
    commit();

    unknowable(Bob, Alice(_handAlice, _saltAlice));
    Bob.only(() => {
      const handBob = declassify(interact.getHand(limit));
      
    });
    Bob.publish(handBob)
      .timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));
    commit();

    Alice.only(() => {
      const saltAlice = declassify(_saltAlice);
      const handAlice = declassify(_handAlice);
    });
    Alice.publish(saltAlice, handAlice)
      .timeout(relativeTime(deadline), () => closeTo(Bob, informTimeout));
    checkCommitment(commitAlice, saltAlice, handAlice);
    
    [stage, hand1, hand2] = [stage-1, handAlice, handBob];
    continue;

  }
```
When any of the conditions of the while loop are met the loop ends and an outcome is generated by passing the right arguments into the winner function. The  output of the winner function is passed into an outcome immutable variable aka const and it is then passed into the House `seeOutcome` interact so the House also gets to see the outcome.  
- Finally to disburse the wager and empty the contract according to the outcome of the game, the `payWinner` function is used to implement this feature and display the outcome to both Alice and Bob.   
  
```js
  //Using the winner function with arguments of the users inputs and the random number to get the winner
  const outcome = winner(rand, hand1, hand2);
  House.interact.seeOutcome(outcome);

//Uses the outcome to pay the winner
  payWinner(outcome,wager, Alice, Bob, House);

  commit();

});
```
While building with reach it is important to track the flow of the money and ensure the contract is empty at the end of the contract to aviod a `balance sufficent for  transfer `  error leading to faliures in the verification process.  
  
Below is the full index.rsh code for your perusual.  
```js
"reach 0.1";
//Outcome array
const [ isOutcome, B_WINS, DRAW, A_WINS ] = makeEnum(3); 

//This computes the winner of the game
const winner = (rand,hand1, hand2) => {
  const x = rand;
  if (hand1 == x && hand2 != x){
    return A_WINS;
  }
  else if(hand1 != x  && hand2 == x){
    return B_WINS;
  }
  else if (hand1 == x && hand2 == x){
    return DRAW;
  }
  else  return DRAW;

};

  
// Makes the required payment to the winner
const payWinner = (outcome, wager, Alice, Bob, House) => {
  if (outcome == DRAW) {
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome)
    });
    transfer(2*wager).to(House);
  }
  else if(outcome == A_WINS) {
    transfer(2*wager).to(Alice);
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome);
    });
  }
  else {
    transfer(2*wager).to(House);
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome);
    });
  }
}
  
const Common  =  {
    ...hasRandom
};
//Player abilities
const Player = {
  seeRules: Fun([UInt, UInt], Null),
  getHand: Fun([UInt], UInt),
  seeOutcome: Fun([UInt], Null),
  informTimeout: Fun([], Null),
  informNewRound: Fun([], Null),

};

  
export const main = Reach.App(() => {
// House Interface
const House = Participant('House', {
    ...Common,
    wager: UInt,
    limit: UInt,
    trials: UInt,
    deadline: UInt,
    rand: Fun([UInt], UInt),
    waitingForAttacher: Fun([], Null),
    seeOutcome: Fun([UInt], Null),
})
//Alice interface
  const Alice = Participant('Alice', {
    ...hasRandom,
    ...Player,
    acceptWager: Fun([UInt], Null), 
    //waitingForAttacher: Fun([], Null)
  });
//Bob interface
  const Bob   = Participant('Bob', {
    ...hasRandom,
    ...Player,
    acceptWager: Fun([UInt], Null),
  });
  init();

  const informTimeout = () => {
    each([Alice, Bob], () => {
      interact.informTimeout();
    });
  };
//Alice and Bob accept the rules and pay the wager
  House.only(() => {
    const limit = declassify(interact.limit);
    const trials = declassify(interact.trials);
    const wager = declassify(interact.wager);
    const deadline = declassify(interact.deadline);
    const rand = declassify(interact.rand(limit));
  });
  House.publish(limit, trials, wager, deadline, rand)
  commit();

  House.interact.waitingForAttacher();
  Alice.only(() => {
    interact.seeRules(limit, trials);
    interact.acceptWager(wager);
  });
  Alice.publish();
  commit();
  Alice.pay(wager);
  commit();

  
  Bob.only(() => {
    interact.seeRules(limit, trials);
    interact.acceptWager(wager);
    
  });
  Bob.publish()
  .pay(wager)
    .timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));

//While loop that loops as long as  the conditions are not met
  var [stage, hand1, hand2 ] = [trials,0,0];
  invariant( balance() == 2 * wager);

  while ((hand1!= rand && hand2 != rand) && (stage > 0)) {
    commit();

    Bob.interact.informNewRound();
    Alice.interact.informNewRound();

    Alice.only(() => {
      const _handAlice = interact.getHand(limit);
      
      const [_commitAlice, _saltAlice] = makeCommitment(interact, _handAlice);
      const commitAlice = declassify(_commitAlice);
      });
    Alice.publish(commitAlice)
      .timeout(relativeTime(deadline), () => closeTo(Bob, informTimeout));
    commit();

    unknowable(Bob, Alice(_handAlice, _saltAlice));
    Bob.only(() => {
      const handBob = declassify(interact.getHand(limit));
      
    });
    Bob.publish(handBob)
      .timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));
    commit();

    Alice.only(() => {
      const saltAlice = declassify(_saltAlice);
      const handAlice = declassify(_handAlice);
    });
    Alice.publish(saltAlice, handAlice)
      .timeout(relativeTime(deadline), () => closeTo(Bob, informTimeout));
    checkCommitment(commitAlice, saltAlice, handAlice);
    
    [stage, hand1, hand2] = [stage-1, handAlice, handBob];
    continue;

  }
  //Using the winner function with arguments of the users inputs and the random number to get the winner
  const outcome = winner(rand, hand1, hand2);
  House.interact.seeOutcome(outcome);

//Uses the outcome to pay the winner
  payWinner(outcome,wager, Alice, Bob, House);

  commit();

});
```
# Conclusion  
    
If you followed this workshop from the beginning to this point, congratulations on building a random number guessing game with a wager using Reach!   
You can decide to level up your game and write your own guessing game smart contract with extra features, or, you can decide to build a GUI for this smart contract and add it to your portfolio. This would be a great way to start as a web3 developer.   
    
For more details on how to build out the GUI for this smart contract you can check out the github [repo](https://github.com/yoru01/The-Price-Is-Right-Casino-Edition).  
