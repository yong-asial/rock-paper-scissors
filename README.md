# Rock Paper Scissors

![demo480](https://user-images.githubusercontent.com/28589278/149097749-a8dfd2e0-c5dc-4afa-9c3c-d3c7166a0317.gif)

## Functionalities

Let's first describe the application layout and functionalities that we are going to build. There are four main parts from the screenshot we have above. The first part is the live video feed from the camera and it will classify the image we are showing to the camera. Next, we have the "Hand" sign as the "Player" and the "Scissors" sign as the computer. The "Hand" sign is what our model classifies from what it sees in the camera and the "Scissors" is a random sign chosen by the computer. Following the two buttons, the first button is the toggle button which can start playing or pausing the game and the second button is a toggle button to whether display the camera or not. Finally, the last part is the result section from the comparison between the player's and the computer's chosen hand.

## Start Project

- Run `npm install` to install project dependencies.
- Run `monaca preview` to preview the app.

## Tutorial

We will cover how to train the machine learning (ML) model with Teachable Machine, incorporate the model with web and mobile applications, and finally create a simple rock-paper-scissors game to play with a computer. In particular, we will be building an image classification model in which we show our hand to the camera, and the model will predict whether the hand is "rock", "paper", or "scissors".

### What is Teachable Machine?
Teachable Machine is a web-based tool that allows us to easily train ML models without much prior knowledge with machine learning. The tool simplifies the process from adding training data, training the model, and exporting the model. You determine what "Classes" you want to train, then add your input with your webcam and hit "Train." The model then quickly learns the difference between these Classes you made. It is pretty easy!

[Read more](https://medium.com/the-web-tub/create-your-first-machine-learning-mobile-application-853176e4eaa1)
