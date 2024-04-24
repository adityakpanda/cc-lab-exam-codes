
## FCFS
---
```java
import java.sql.*;

// 4 private strings and 3 private int
// connection 
// cut 5th step
// cloudlet list creation 
// broker submission 
// function declaration
/*
cloudletList = new ArrayList<Cloudlet>();

			// Cloudlet properties
			int id = 0;
			long length = 400000;
			long fileSize = 300;
			long outputSize = 300;
			UtilizationModel utilizationModel = new UtilizationModelFull();

			Cloudlet cloudlet = new Cloudlet(id, length, pesNumber, fileSize, outputSize, utilizationModel, utilizationModel, utilizationModel);
			cloudlet.setUserId(brokerId);
			cloudlet.setVmId(vmid);

			// add the cloudlet to the list
			cloudletList.add(cloudlet);

			// submit cloudlet list to the broker
			broker.submitCloudletList(cloudletList);
*/

private static final String url = "jdbc:mysql://localhost:3306/cc";
private static final String id = "root";
private static final String pass = "root";
private static final String query1 = "SELECT id , length FROM cloudletprop";

private static int vmid;
private static int pesNumber;
private static int brokerId;
// remove int for the further declarations in the file

// enter into main function
Connection connection = DriverManager.getConnection(url , id , pass);
cloudletList = retrieve(connection); // first line in 5th step 
broker.submitCloudletList(cloudletList); // last line in 5th step 

// outside the main function 
private static List<Cloudlet> retrieve(Connection connection) // return type is same as datatype of cloudletList
{
	List<Cloudlet> cloudlets = new ArrayList<>();	
	// Cloudlet properties except id and length
	long fileSize = 300;
	long outputSize = 300;
	UtilizationModel utilizationModel = new UtilizationModelFull();

	try(Statement statement = connection.createStatment();
	Resultset resultset = statement.executeQuery(query1))
	{
		while(resultset.next())
		{
			int id = resultset.getInt("id");
			int length = result.getLong("length");
			Cloudlet cloudlet = new Cloudlet(id, length, pesNumber, fileSize, outputSize, utilizationModel, utilizationModel,   utilizationModel);
			cloudlet.setUserId(brokerId);
			cloudlet.setVmId(vmid);
			// add the cloudlet to the list
			cloudlets.add(cloudlet); // use cloudlets not cloudletList
		}
	}
	catch(SQLException e)
	{
		e.printstackTrace();
	}
	return cloudlets;
}

```

## max-min

```java 
import java.sql.*;
import java.util.Comparator;


private static final String JDBC_URL = "jdbc:mysql://localhost:3306/cc";
private static final String DB_USER = "root";
private static final String DB_PASSWORD = "root";
private static final String SELECT_VM_DATA = "SELECT id, mips FROM VMProperties";
private static final String SELECT_CLOUDLET_DATA = "SELECT id, length FROM CloudletProperties";

private static List<Cloudlet> cloudletList;
private static List<Vm> vmList;

private static int pesNumber;
private static int brokerId;

// ---------------remove entire 4th step------------------------------ 
/* 
// VM description
			vmid = 0;
			int mips = 1000;
			long size = 10000; // image size (MB)
			int ram = 512; // vm memory (MB)
			long bw = 1000;
			pesNumber = 1; // number of cpus
			String vmm = "Xen"; // VMM name

			// create VM
			Vm vm = new Vm(vmid, brokerId, mips, pesNumber, ram, bw, size, vmm, new CloudletSchedulerTimeShared());

			// add the VM to the vmList
			vmlist.add(vm);

			// submit vm list to the broker
			broker.submitVmList(vmlist);

*/
Connection connection = DriverManager.getConnection(JDBC_URL, DB_USER, DB_PASSWORD);

vmList = createVMList(connection);
vmList.sort(Comparator.comparingDouble(Vm::getMips));
broker.submitVmList(vmList);

cloudletList = retrieveCloudletProperties(connection);
cloudletList.sort((c1, c2) -> Long.compare(c2.getCloudletLength(), c1.getCloudletLength()));
assignCloudletsToVMs(cloudletList, vmList);
broker.submitCloudletList(cloudletList);

//--------------------------------------------------------------------------------------
private static List<Cloudlet> retrieveCloudletProperties(Connection connection) {
        List<Cloudlet> cloudlets = new ArrayList<>();
        long fileSize = 300;
        long outputSize = 300;
        UtilizationModel utilizationModel = new UtilizationModelFull();

        try (Statement statement = connection.createStatement();
             ResultSet resultSet = statement.executeQuery(SELECT_CLOUDLET_DATA)) {

            while (resultSet.next()) {
                int id = resultSet.getInt("id");
                long length = resultSet.getLong("length");

                // Create Cloudlet object and set properties
                Cloudlet cloudlet = new Cloudlet(id, length, pesNumber, fileSize, outputSize,
                        utilizationModel, utilizationModel, utilizationModel);
                cloudlet.setUserId(brokerId);

                // Add the cloudlet to the list
                cloudlets.add(cloudlet);
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }

        return cloudlets;
    }

//--------------------------------------------------------------------------------------
private static List<Vm> createVMList(Connection connection) {
        List<Vm> vms = new ArrayList<>();

        try (Statement statement = connection.createStatement();
             ResultSet resultSet = statement.executeQuery(SELECT_VM_DATA)) {

            while (resultSet.next()) {
                int id = resultSet.getInt("id");
                int mips = resultSet.getInt("mips");

                // VM description
                int vmId = id;
                long size = 10000; // image size (MB)
                int ram = 512; // vm memory (MB)
                long bw = 1000;
                pesNumber = 1; // number of cpus
                String vmm = "Xen"; // VMM name

                // Create VM
                Vm vm = new Vm(vmId, brokerId, mips, pesNumber, ram, bw, size, vmm, new CloudletSchedulerSpaceShared());

                // Add the VM to the list
                vms.add(vm);
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }

        return vms;
    }

//--------------------------------------------------------------------------------------
private static void assignCloudletsToVMs(List<Cloudlet> cloudlets, List<Vm> vms) {
    for (int i = 0; i < cloudlets.size(); i++) {
        Cloudlet cloudlet = cloudlets.get(i);
        Vm vm = vms.get(i);
        cloudlet.setVmId(vm.getId());
    }
}
```


## MySQL code 
---
```sql
Create database cc;

Use cc;

-- create table to store id and length
CREATE TABLE CloudletProperties (
    id INT PRIMARY KEY AUTO_INCREMENT,
    length INT
);

-- Insert 5 entries with increasing order of id and jumbled values for length
INSERT INTO CloudletProperties (length) VALUES
    (500),
    (1000),
    (300),
    (800),
    (600);

-- Create VM properties table
CREATE TABLE VMProperties (
    id INT AUTO_INCREMENT PRIMARY KEY,
    mips INT
);

-- Insert sample data into VMProperties table
INSERT INTO VMProperties (id, mips) VALUES
(1, 800),
(2, 900),
(3, 700),
(4, 950),
(5, 850);
```



