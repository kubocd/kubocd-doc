# The KuboCD Package

Required attributes are flagged with '*'

## Package object:

### **apiVersion*

- Manifest version. Only allowed value is currently `v1alpha1`

### **type*

- Manifest type. Only allowed value is `Package`. May be omitted.

### **name*

- The name of the package. Will be used in the OCI repository name.

    dlmkdld mlkm dlmldkm

### - *tag

lkk lkl lk lk lk ll l

### description
### protected
### schema.parameters
### schema.context
### *modules
### roles
### dependencies


## Module object

### *name
### *source
### values
### timeout
### enabled
### targetNamespace
### dependsOn
### specPatch

## HelmRepository source

### url
### chart
### version

## OCI source

### repository
### tag
### insecure

## Git source

### url
### branch
### tag
### path

## Local source

### path
