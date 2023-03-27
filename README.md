----
Author : NDIAYE Mansour

Context : Bootcamp DevOps training, 12nd promotion

Training center : eazytraining.fr

Date : 24th March 2023

Linkedin : https://www.linkedin.com/in/mansour-ndiaye-64a04b161/

---

# Kubernetes mini - project
The aim of this project is to deploy wordpress and mysql in a kubernetes cluster following these steps : 
- Create a mysql deployment with only one replica,
- Create a service type clusterIP to expose mysql pods
- Create a wordpress deployment with all environmental variables to log into the mysql database
- Create a service type node port to expose the frontend wordpress

## Architecture

![image](https://user-images.githubusercontent.com/58290325/228014858-f2420df6-099b-4e57-8f93-85e8fecd8a89.png)


## Step 1 : Namespace creation

