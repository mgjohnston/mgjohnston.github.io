---
layout: post
title: 538 Riddler - Can You Get A Haircut Already?  
categories: [coding]
tags: [538riddler, R]
---
From Dave Moran comes a question we’ve all faced at some point when waiting in line for a haircut:

At your local barbershop, there are always four barbers working simultaneously. Each haircut takes exactly 15 minutes, and there’s almost always one or more customers waiting their turn on a first-come, first-served basis.

Being a regular, you prefer to get your hair cut by the owner, Tiffany. If one of the other three chairs opens up, and it’s your turn, you’ll say, “No thanks, I’m waiting for Tiffany.” The person behind you in line will then be offered the open chair, and you’ll remain at the front of the line until Tiffany is available.

Unfortunately, you’re not alone in requesting Tiffany — a quarter of the other customers will hold out for Tiffany, while no one will hold out for any of the other barbers.

One Friday morning, you arrive at the barber shop to see that all four barbers are cutting hair, and there is one customer waiting. You have no idea how far along any of the barbers is in their haircuts, and you don’t know whether or not the customer in line will hold out for Tiffany.

What is the expected wait time for getting a haircut from Tiffany?

<https://fivethirtyeight.com/features/can-you-get-a-haircut-already/>

### The Answer

14 minutes 3 seconds. 

### The Code

  ```R
  set.seed(1)
waitTime<-function(){
  tiffany<- runif(1)*15
  barber<- runif(3)*15
  tiffanyWanted <- sample(c(T,F,F,F),1)
  
  if (tiffanyWanted){
    time <- tiffany + 15
  }
  else(
    if (all(tiffany < barber)){
      time <- tiffany + 15
    }
    else{
      time <- tiffany
      
    }
  )
  return(time)
}
meanWait <- NULL
for(j in 1:1000){
waitTimed<-NULL
for (i in 1:1000){
  waitTimed<-rbind(waitTimed,waitTime())
}
meanWait<-c(meanWait,mean(waitTimed))
}
mean(meanWait)
hist(meanWait)
```
