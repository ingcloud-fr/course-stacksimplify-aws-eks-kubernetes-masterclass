# YAML Basics

## Step-01: Comments & Key Value Pairs
- Space after colon is mandatory to differentiate key and value
```yml
# Defining simple key value pairs
name: kalyan
age: 23
city: Hyderabad
```

## Step-02: Dictionary / Map
- Ensemble de propriétés regroupées après un élément.
- En YAML, un dictionnaire ou une map (abréviation de mapping) est une structure de données qui stocke des paires clé-valeur
- Quantité égale d'espaces vides requise pour tous les éléments d'un dictionnaire.
```yml
person:
  name: kalyan
  age: 23
  city: Hyderabad
```

## Step-03: Array / Lists
- Le tiret indique un élément d'un tableau
```yml
person: # Dictionary
  name: kalyan
  age: 23
  city: Hyderabad
  hobbies: # List  
    - cycling
    - cookines
  hobbies: [cycling, cooking]   # List with a differnt notation  
```  

## Step-04: Multiple Lists
- Le tiret indique un élément d'un tableau
```yml
person: # Dictionary
  name: kalyan
  age: 23
  city: Hyderabad
  hobbies1: # Liste avec tirets (notation traditionnelle pour les listes)
    - cycling
    - cooking
  hobbies2: [cycling, cooking]  # Liste avec notation compacte (style JSON)
  friends: # Liste de dictionnaires (maps)
    - name: friend1
      age: 22
    - name: friend2
      age: 25            
```  


## Step-05: Sample Pod Template for Reference
```yml
apiVersion: v1 # String
kind: Pod  # String
metadata: # Dictionary
  name: myapp-pod
  labels: # Dictionary 
    app: myapp         
spec:
  containers: # List
    - name: myapp
      image: stacksimplify/kubenginx:1.0.0
      ports: # Liste de dictionnaires (maps)
        - containerPort: 80
          protocol: "TCP"
        - containerPort: 81
          protocol: "TCP"
```




