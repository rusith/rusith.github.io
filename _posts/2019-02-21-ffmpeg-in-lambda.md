---
layout: post
title: Running FFmpeg in AWS Lambda using NodeJS
tags: aws lambda
comments: true
---
Okay, no drama. let’s get right to the point.

When you have a requirement to do something on video/ audio files, the tool of choice would be FFmpeg which is free and open source.

What if you want to run FFmpeg from a Lambda function?

In that case, you will face some problems,

1. Need to find an executable Which works on the Lambda server.
2. Copy the FFmpeg to the `/tmp` folder when you are going to execute it because you can’t execute anything from anywhere other than `/tmp` folder in Lambda.
3. You will have to give necessary permissions to execute FFmpeg from `/tmp` folder


## Finding a build

To run FFmpeg you have to have a build that is compatible with the Lambda environment. after much searching, I found this **[website](https://johnvansickle.com/ffmpeg/)** by this [cool dude](https://www.patreon.com/johnvansickle), Where he has listed a set of static builds of FFmpeg for different environments. You can download [FFmpeg-release-amd64-static.tar.xz](https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz) file which is the one matches the Lambda machine and extract it inside your Lambda function package.

The package folder will look like something like this after extracting FFmpeg

<img src="{{ site.url }}/public/images/2019-02-21-ffmpeg-in-lambda/file-structure.png">

## Running It

You will need to have chmod and ncp installed to run this code. Run `npm install chmod ncp — save` to install the packages.

Let’s jump right into a code which executes FFmpeg.

```js
const exec = require('child_process').execFile
const { ncp } = require('ncp')
const path = require('path')
const chmod = require('chmod')

exports.mux = async (audioFile, videoFile, muxedFile) => new Promise((resolve, reject) => {
  /*
    We can't run the ffmpeg from the current folder because it is read only.
    in Lambda all locations except /tmp are read only.
    So I had to copy the ffmpeg to the temp folder and give it execute permissions to make it work
   */
  // ncp will copy the whole directory to /tmp/ffmpeg
  ncp(path.join(__dirname, '../ffmpeg/'), '/tmp/ffmpeg', {
    clobber: false, // Do not overwrite if already exists (Lambda sometimes re-uses the container)
  }, (err) => {
    if (err) {
      reject(err)
    } else {
      // Give execute permissions to the ffmpeg
      chmod('/tmp/ffmpeg/ffmpeg', {
        execute: true,
      })
      
      // Muxing an audio file and a video file into one video file
      // This could be any operation on FFmpeg, you just have to give the parameters correctly.
      exec('/tmp/ffmpeg/ffmpeg', ['-y', '-i', videoFile, '-i', audioFile, '-map', ' 0:0', '-map', '1:0', '-c', 'copy', muxedFile], {}, (error, stdout) => {
        if (error) {
          reject(error)
        } else {
          resolve(stdout)
        }
      })
    }
  })
})
```

In here I am trying to merge two files into one. This could be any other operation. The first thing we do here is copying the FFmpeg folder to the tmp folder, then we give the execute permission to the FFmpeg file using the chmod node package.

You get the idea. You can even speed up the process by just copying the archive itself into the package and extracting it to the temp folder in Lambda.