### Azure Devops + Internal Nuget server for powershell module publishing

Recently, I found myself needing to distribute code to any windows server host in our environment such that any relevant account local to any host would be able to reliably run my code. Before traversing the internet to figure out how to do this, I considered a couple of low hanging options -

1. Home the modules in some network-accessible file share
2. Just push the module out to each target host via work pipeline
    * generic psremoting, ansible, etc
3. Push module to public psgallery (lol)
4. Home modules in fileshare and deploy to all targets via gpo

The first option, while valid, didn't feel robust enough for our environment - versioning would be dependent on a local git repo at best, and I could not reliably pin old versions without duplicating repo version structure in persistent storage.  

The second option would rely on either writing all module files to the disk of the target host each time, which would be difficult to enforce in a standard way across all hosts, or it would require pushing the code out and defining all the functions / importing module at runtime for each target host, which would increase execution time of the code against all remote hosts (by a lot) + require more bandwidth + require more runtime complexity + just doesn't feel like the best option.  

Using public PSGallery in third option was a real idea - if the code is generic enough, and org is okay with it, you could totally publish to psgallery and be done with it. But, this would introduce internet connectivity requirements for all hosts which much install and update the module on every run. It would also introduce a few annoyances with how clean the code would have to be to be uploadable publicly.  

GPOs are slow and fail often without much in the way of centralized alerting. Similarly, doing something via intune would also feel a bit shoddy. Too hard to make sure all hosts have updated code available with some push method like this. Much more challenging to make a code change, commit, push to main, and then under a minute later successfully run the code against all hosts.  

## Enter nuget server

While old and with docs that leave something to be desired, the internal nuget server that Microsoft still officially supports is a fantastic way to achieve PSGallery functionality for internal network. It is also very straightforward to hook up to Azure Devops for end-to-end publishing on commits. Kevin Marquette wrote a great [blogpost](https://powershellexplained.com/2018-03-03-Powershell-Using-a-NuGet-server-for-a-PSRepository/) several years ago which made this dead simple for me to set up. Here are the relevant msft learn links -  
This one documents setting up and publishing the server -  
https://learn.microsoft.com/en-us/powershell/gallery/how-to/working-with-local-psrepositories?view=powershellget-3.x  

This one documents creating the actual asp.net core nuget.server app (up to the point of publishing packages per the first article, where it is linked) -  
https://learn.microsoft.com/en-us/nuget/hosting-packages/nuget-server  

## get visual studio

On windows, install visual studio community edition like this if you don't have it, (or enterprise by swapping community -> enterprise) like this:

    winget install --id Microsoft.VisualStudio.2022.Community

Once that is installed, follow the instructions in the nuget-server msft learn link. I thought this would be easy myself, but I had never published an application via visual studio before, so got a little stuck on step 9, "Once you've tested your local deployment, deploy the application to any other internal or external site as needed.". Uh, draw the rest of the owl much? Kidding aside I really had no idea what to do, and copying the files from my workstation to a server, I ran into 500 error when launching the site via iis first time, so I had to mess around with it a bit until I figured it out. 
# todo : finish drawing rest of owl
# rest of owl

Ok. I kind of brute force followed docs and built with gui. It is probably better to build automatically somehow.
