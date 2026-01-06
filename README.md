# Automate the guestbook







* Everything worked in the github action and everything was deployed onto Openshift cluster, but I could not open the site. 
Problem: Oc route requires a named Service port.
Fix: need to add a name to the frontend  service port & also note it down into the Route yaml file
