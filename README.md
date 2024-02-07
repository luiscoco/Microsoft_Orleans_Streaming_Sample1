# Orleans Streaming with Azure EventHub (.NET8 Client-Server)

## 1. Create an EventHub in Azure

## 2. Create a Table and Storage Account in Azure

We can see the overview storage account

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/86438bb1-52ca-48cb-8e2f-7c4193d8ffff)

We copy the 

## 3. Create a blank solution in Visual Studio 2022 Community Edition


## 4. Create the Server(Silo) console project


## 5. Create the Client application console project


## 6. Create the GrainIntefaces 


## 7. Create the Grains


## 8. Create the Secrets.json file

```json
{
  "EventHubConnectionString": "",
  "DataConnectionString": ""
}
```

We place the **Secrets.json** file in SiloHost and Client projects root. See the following picture

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/faa81924-fd07-462e-a3ce-169405875b2a)

## 9. Run and Test the solution

We right click on the solution name and we select the menu option "**Set the startup projects**" 

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/d0985adc-d76b-4a84-8cee-f9b8a44b7852)

We set the SiloHost and Client as startup projects

![image](https://github.com/luiscoco/Microsoft_Orleans_Streaming_Sample1/assets/32194879/09deaae0-0097-42e3-87e8-ae76b2b2a8ed)

We build and run the solution. Two console windows are automatically are opened and the data is interchanged between the Server and the Client



