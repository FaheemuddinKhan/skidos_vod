# eskidos_vod

### Video On Demand Service Assignment - HLD and LLD for a Video on Demand backend microservice to serve fun learning videos

#### Requirements

1. Only authenticated users should have access to the videos
2. Video microservice should support adaptive bitrate streaming
3. Video microservice should support encryption of video
4. Video search feature

### High Level Design of VoD Service


#### Upload Microservice

![2022-07-31](https://user-images.githubusercontent.com/37400411/182039310-2cb57c42-b7d4-4752-afd4-5b83fc22ebdc.png)



The upload microservice will be triggered while creating the content on this application. This also helps in achieving requirement 2 shown above which is adaptive bitrate streaming. We will talk about it later in this document. 

When the uploader hits the upload file or create request on the API gateway, the request will be redirected to the Load balancer which will choose the upload service instance available from the cluster. The upload service will process the file and send the media file to the video encoder tool which will further process the video and output the number of files of different resolutions and also the encryption will be done at this point. Once done, the upload service will store the output files to File Storage/Video Storage Database and the metadata will be stored in SQL database. Also, once the encoded files are stored in File Storage, it will also be sent to the CDN and the CDN URL will be persisted in MetaData Storage.

#### Video Microservice

![2022-08-01](https://user-images.githubusercontent.com/37400411/182084007-cbdf2050-fa3a-49a0-bba6-77f986d00313.png)

When the end user makes a view request for a video or searches for any video using keywords, the request will be redirected to the Video microservice. It interacts with Metadata storage to fetch the information especially nearest CDN with video Url and responds back with that information and then the user makes request to the nearest CDN node to view the video. For Content discovery, video microservice interacts with the Elastic search and metadata storage to send back the matched video and the simailar results in the response.
