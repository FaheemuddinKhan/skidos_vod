# eskidos_vod

### Video On Demand Service Assignment - HLD and LLD for a Video on Demand backend microservice to serve fun learning videos

#### Requirements

1. Only authenticated users should have access to the videos
2. Video microservice should support adaptive bitrate streaming
3. Video microservice should support encryption of video
4. Video search feature

### High Level Design of VoD Service


Upload Microservice

![2022-07-31](https://user-images.githubusercontent.com/37400411/182038977-aa6f7e92-0305-45ae-8efe-a58edf9d83d7.png)

The upload microservice will be triggered while creating the content on this application. This also helps in achieving requirement 2 shown above which is adaptive bitrate streaming. We will talk about it later in this document. 

When the uploader hits the upload file or create request on the API gateway, the request will be redirected to the Load balancer which will choose the upload service instance available from the cluster. The upload service will process the file and send the media file to the video encoder tool which will further process the video and output the number of files of different resolutions and also the encryption will be done at this point. Once done, the upload service will store the output files to File Storage/Video Storage Database and the metadata will be stored in SQL database. Also, once the encoded files are stored in File Storage, it will also be sent to the CDN and the CDN URL will be persisted in MetaData Storage.

