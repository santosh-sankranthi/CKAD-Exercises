## resources that extend k8s 

There are 3 objects in this topic:-
1. CRD
2. CR
3. operator

what will you be asked in the exame?

1. how to discover existing CRs or CRDs
2. how to create CRs or CRDs
3. how to troubleshoot CRs or CRDs
4. how to install an operator using helm.


-----------------------DISCOVERING CRDs CRs---------------------------------

discover crd: k get crds

getting the resources of a specific type :- k get <custom-resource>

k explain <crd-name>

k api-resources | grep <resource-name>  [case sensitive]

k api-resources | grep -i <resource-name> [case insensitive]

kubectl get <cr-type> -A (get all the cr resources created acroass all ns)

also an operator can have multiple CRDS, 

- how to find out all the CRs that an operator is managing ?

get the domain of the operator (operator would be a pod running at a place) using describe command, and do this commmand :- kubectl api-resorces | grep -i <domain name>

-----------------------creating CRDs CRs---------------------------------

creating a CRD:-

search for custom resource, scroll to the bottom most copy paste the same template and make the following changes:

1. first find the group and if it is namespaced or not
2. find the singular plural, if not given keep yours
3. update metadata.name in this format: pluralname.group
4. update the spec accordingly
5. create teh crd

sometimes a shortname should also be given, but that parameter wont be there in the template so add it as a dictionary at the singular level.

creating a CR:-

first exe "k explain <crd-name>" and then see its group and version and kind.

copy paste the pod template and replace those values first,

and then replace the rest of the values like spec etc. and then verify everything is correct, spelling mistakes can happen so be careful.

---------------------------TROUBLESHOOTING CKAD STUFF-----------------------------------------

since you already know how to create crs, and crds, the only thing that you can do to clear this place is, use error message and proceed from there not vaguely.

---------------------------OPERATORS-----------------------------------------


If you get a troubleshooting question, it will look like this:

Mock Exam Question:

"A developer created an AppScaler custom resource named backend-scaler in the dev namespace, but the application is not scaling.
Locate the operator managing this resource in the ops-system namespace, find out why it is failing, and fix the custom resource so the operator can successfully scale the app."

How you solve it (Focusing on Speed):

You do NOT look at the CR first. You go straight to the Operator.
Find the pod: kubectl get pods -n ops-system
Check the Operator's brain (the logs): kubectl logs <operator-pod-name> -n ops-system
The logs will say something like: Error processing AppScaler 'backend-scaler': field 'max-replicas' must be an integer, got string.
You edit the CR (kubectl edit appscaler backend-scaler -n dev), fix the typo, and save.

