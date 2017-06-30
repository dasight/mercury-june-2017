# Streaming workshop:

Task: get the generator up and running.

* Step 1: 
Java projects received. 

* Step 2: 
Need to compile the java program. In order to do that, I installed IntelliJ on my Mac. 
I needed to add an additional idk on my Mac, because the cluster is running a idk 1.7

In IntelliJ, I needed to figure out how to compile the .java source and project into a .jar file. Google was helpful here.

Project compiled and transferred to 1 host in the cluster.

The generator starts with the following command:

/usr/java/jdk1.7.0_67-cloudera/bin/java -cp bootcamp-0.0.1-SNAPSHOT.jar com.cloudera.fce.bootcamp.MeasurementGenerator ip-172-31-47-120.us-west-2.compute.internal 9092

---

Added parameter to the generator, to wait for some miliseconds.

The new MeasurementGenerator class now looks as follows:


```
public class MeasurementGenerator {

    @SuppressWarnings("resource")
    public static void main(String[] args) throws Exception {

        String hostName = args[0];
        int portNumber = Integer.parseInt(args[1]);
        int waitTime = Integer.parseInt(args[2]);
        Socket echoSocket = new Socket(hostName, portNumber);
        PrintWriter out = new PrintWriter(echoSocket.getOutputStream(), true);

        System.out.println("Measurement Generator Tool");
        System.out.println("  HostName       : " + hostName);
        System.out.println("  Port Number    : " + portNumber);
        System.out.println("  Wait Time (ms) : " + waitTime);



        while (true) {
            Random random = new Random();
            
            String measurementID = UUID.randomUUID().toString();
            int detectorID = random.nextInt(8) + 1;
            int galaxyID = random.nextInt(128) + 1;
            int astrophysicistID = random.nextInt(106) + 1;
            long measurementTime = System.currentTimeMillis();
            double amplitude1 = random.nextDouble();
            double amplitude2 = random.nextDouble();
            double amplitude3 = random.nextDouble();
            
            String delimiter = ",";
            String measurement = measurementID + delimiter + detectorID + delimiter + galaxyID +
                    delimiter + astrophysicistID + delimiter + measurementTime + delimiter +
                    amplitude1 + delimiter + amplitude2 + delimiter + amplitude3;
            
            out.println(measurement);
            System.out.println(measurement);

            Thread.sleep(waitTime);

        }
        
    }

}
```

The new call is: 

/usr/java/jdk1.7.0_67-cloudera/bin/java -cp bootcamp-0.0.1-SNAPSHOT.jar com.cloudera.fce.bootcamp.MeasurementGenerator ip-172-31-47-120.us-west-2.compute.internal 9092 5000

This will wait 5 seconds before a new record is generated.

---

