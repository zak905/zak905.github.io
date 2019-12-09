---
layout: post
title: "Extracting memory usage from docker stats using bash tricks"
author: "Zakaria"
comments: true
description: docker stats
---


`docker stats` is a docker cli command that provides resource usage statistics for the current running containers. `docker stats` provides only the current time data, and therefore it is necessary to record that data if one would like to plot it or keep it for doing some analysis. In this post, I would like to share some simple bash tricks for recording the resource usage data.

## Basic usage

`docker stats` streams the resource usage of all the current running containers, the output of the command looks like:

```
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
79cf60f4e58f        container_1          0.03%               12.32MiB / 15.53GiB   0.08%               8.59kB / 0B         33.4MB / 48.3MB     7
79cf60f4e58f        container_2          0.03%               12.32MiB / 15.53GiB   0.08%               8.59kB / 0B         33.4MB / 48.3MB     7

```

if you would like to get the usage at the current point of time without streaming then the `--no-stream` option should be used.

either case the historical data is lost. If we would like to analyze resource usage patterns, the data is to be recorded. 

## Filtering and recording the data

As an initial step, we can use `docker  stats --no-stream >> data.txt`, but we will have to run this command manually several times on a regular interval to get a meaningful data set. 

To improve further, we can wrap the command into a loop and interrupt manually the process (using `kill` or `Control Key + C`) whenever we want to end the recording: `while true; do docker stats --no-stream >> data.txt; done`

However, the output returns data for all the containers so we can filter by container name: `while true; do docker stats --no-stream | grep container1 >> data.txt; done`. This will display only the data row without header, which takes us even closer to what we want to achieve. 

```
79cf60f4e58f        container_1          0.03%               12.32MiB / 15.53GiB   0.08%               8.59kB / 0B         33.4MB / 48.3MB     7
```

Now, we will need to extract only the values that we care about. In our case, we are aiming for `MEM USAGE` value.

A nice bash oldie is the `awk` which provides several functionalities for text processing and filtering. Since our `MEM USAGE` is in the 4th column, we need to add the following pipe to our command so that it diplays only the memory value: `awk '{print $4}'`, thus our command looks like:

`while true; do docker stats --no-stream | grep container1 | awk '{print $4}' >> data.txt; done`

Output:

```
12.2MiB
12.2MiB
12.2MiB
```

Finally, to get the numbers only without the unit we can chop off the `MiB` using a utility like `sed`: 

`while true; do docker stats --no-stream | grep container1 | awk '{print $4}' | sed -e 's/MiB//g' >> data.txt; done`

Sweet! Now that we have the raw data so we can analyze it or plot it as we want.  

Ps: it's possible to print the data to standard output by replacing the file pipe by a printing function like `echo`, the command would look like then: 

`while true; do docker stats --no-stream | grep container1 | awk '{print $4}' | sed -e 's/MiB//g' | xargs echo ; done`

## Using the data

The data recorded can be ploted as a line graph which is usually the data representation used for memory usage. For this puprpose, there are plenty of free tools like [LibreOffice Calc](https://www.libreoffice.org/discover/calc/) for creating charts based on datasets. 

It's also possible to plot the data directly in terminal using a shell utility like `gnuplot`, so that analyzing the data can be done without leaving the terminal. For example we can use the following gnuplot file to read and refresh the data from our `data.txt` file in real time:

```
plot "data.txt" with lines
pause 1
reread
```

We can save the file as `docker-stats.gnu` and then run `gnuplot docker-stats.gnu` from a different terminal tab. We can now see the memory changes in real time:  

![OAuth approval](https://s3-eu-west-1.amazonaws.com/gwidgets/gnuplot.png)

## Adding the date:

If the amount of memory alone is not enough, and you need to record the time/date when the value was recorded, it's possible to use the bash `date` function and append it to the `awk` print command in the following way: 

`while true; do docker stats --no-stream | grep container1 | awk -v date="$(date +%T)" '{print $4, date}' | sed -e 's/MiB//g' >> data.txt; done`

```
12.2 17:29:35
12.2 17:29:37
12.2 17:29:39
12.2 17:29:41
12.2 17:29:44
12.2 17:29:46
```

## The GiB issue: 

Sometimes the unit of memory juggles between `MiB` and `GiB`. In this case, we must, in addition to removing the units, convert to the desired unit. Let's suppose we want to convert the values displayed in `GiB` to megabytes, we should be able to do the conversion with the help of `awk conditionals`:

`while true; do sudo docker stats --no-stream | grep container1 | awk '{ if(index($4, "GiB")) {gsub("GiB","",$4); print $4 / 1000} else {gsub("MiB","",$4); print $4}}' >> data.txt; done`

Explanation: 

using `index($4, "GiB")` we checked if `GiB` is present in the current value, if it is case then we remove the unit using `gsub("GiB","",$4)` and then we print the value multiplied by 1000 (no need to use `sed` in this case). Otherwise, we remove the `MiB` and print the values as-is. 


## Wrap up: 

using bash pipes and basic functions can be an alternative to using `cpanel` or the other complex tools like `Prometheus` for simple local machine usage situations. 