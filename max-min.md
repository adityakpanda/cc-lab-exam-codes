```java
package org.cloudbus.cloudsim.examples;

import java.text.DecimalFormat;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.LinkedList;
import java.util.List;
import java.sql.*;
import java.util.Comparator;
import org.cloudbus.cloudsim.Cloudlet;
import org.cloudbus.cloudsim.CloudletSchedulerSpaceShared;
import org.cloudbus.cloudsim.Datacenter;
import org.cloudbus.cloudsim.DatacenterBroker;
import org.cloudbus.cloudsim.DatacenterCharacteristics;
import org.cloudbus.cloudsim.Host;
import org.cloudbus.cloudsim.Log;
import org.cloudbus.cloudsim.Pe;
import org.cloudbus.cloudsim.Storage;
import org.cloudbus.cloudsim.UtilizationModel;
import org.cloudbus.cloudsim.UtilizationModelFull;
import org.cloudbus.cloudsim.Vm;
import org.cloudbus.cloudsim.VmAllocationPolicySimple;
import org.cloudbus.cloudsim.VmSchedulerTimeShared;
import org.cloudbus.cloudsim.core.CloudSim;
import org.cloudbus.cloudsim.provisioners.BwProvisionerSimple;
import org.cloudbus.cloudsim.provisioners.PeProvisionerSimple;
import org.cloudbus.cloudsim.provisioners.RamProvisionerSimple;

public class MinMax {

    // Database credentials
    private static final String JDBC_URL = "jdbc:mysql://localhost:3306/cc";
    private static final String DB_USER = "root";
    private static final String DB_PASSWORD = "root";
    private static final String SELECT_VM_DATA = "SELECT id, mips FROM VMProperties";
    private static final String SELECT_CLOUDLET_DATA = "SELECT id, length FROM CloudletProperties";

    private static List<Cloudlet> cloudletList;
    private static List<Vm> vmList;
    private static int pesNumber;
    private static int brokerId;

    public static void main(String[] args) {

        Log.printLine("Starting CloudSimExample1...");

        try {
            int num_user = 1;
            Calendar calendar = Calendar.getInstance();
            boolean trace_flag = false;

            // Initialize the CloudSim library
            CloudSim.init(num_user, calendar, trace_flag);

            // Create Datacenter
            Datacenter datacenter0 = createDatacenter("Datacenter_0");

            // Create Broker
            DatacenterBroker broker = createBroker();
            brokerId = broker.getId();

            // Create VMs
            vmList = createVMList();

            // Sort VMs based on MIPS in ascending order
          vmList.sort(Comparator.comparingDouble(Vm::getMips));


            // Submit VM list to the broker
            broker.submitVmList(vmList);

            // Create Cloudlets
            Connection connection = DriverManager.getConnection(JDBC_URL, DB_USER, DB_PASSWORD);
            cloudletList = retrieveCloudletProperties(connection);

            // Sort Cloudlets based on process length in descending order
            cloudletList.sort((c1, c2) -> Long.compare(c2.getCloudletLength(), c1.getCloudletLength()));

            // Assign Cloudlets to VMs based on MIPS and process length
            assignCloudletsToVMs(cloudletList, vmList);

            // Submit cloudlet list to the broker
            broker.submitCloudletList(cloudletList);

            // Start the simulation
            CloudSim.startSimulation();

            CloudSim.stopSimulation();

            // Print results when simulation is over
            List<Cloudlet> newList = broker.getCloudletReceivedList();
            printCloudletList(newList);

            Log.printLine("CloudSimExample1 finished!");
        } catch (Exception e) {
            e.printStackTrace();
            Log.printLine("Unwanted errors happen");
        }
    }

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

    private static List<Vm> createVMList() {
        List<Vm> vms = new ArrayList<>();

        try (Connection connection = DriverManager.getConnection(JDBC_URL, DB_USER, DB_PASSWORD);
             Statement statement = connection.createStatement();
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

    private static Datacenter createDatacenter(String name) {
        // Creating Host with its id and list of PEs
        List<Host> hostList = new ArrayList<>();
        List<Pe> peList = new ArrayList<>();
        peList.add(new Pe(0, new PeProvisionerSimple(5000))); // Increase MIPS capacity to 5000
        hostList.add(new Host(0, new RamProvisionerSimple(2048), new BwProvisionerSimple(10000), 1000000, peList, new VmSchedulerTimeShared(peList)));

        // Creating DatacenterCharacteristics object
        String arch = "x86"; // system architecture
        String os = "Linux"; // operating system
        String vmm = "Xen";
        double time_zone = 10.0; // time zone this resource located
        double cost = 3.0; // the cost of using processing in this resource
        double costPerMem = 0.05; // the cost of using memory in this resource
        double costPerStorage = 0.001; // the cost of using storage in this resource
        double costPerBw = 0.0; // the cost of using bw in this resource
        LinkedList<Storage> storageList = new LinkedList<>(); // we are not adding SAN devices by now
        DatacenterCharacteristics characteristics = new DatacenterCharacteristics(arch, os, vmm, hostList, time_zone, cost, costPerMem, costPerStorage, costPerBw);

        // Creating Datacenter object
        Datacenter datacenter = null;
        try {
            datacenter = new Datacenter(name, characteristics, new VmAllocationPolicySimple(hostList), storageList, 0);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return datacenter;
    }

    private static DatacenterBroker createBroker() {
        DatacenterBroker broker = null;
        try {
            broker = new DatacenterBroker("Broker");
        } catch (Exception e) {
            e.printStackTrace();
        }
        return broker;
    }

private static void assignCloudletsToVMs(List<Cloudlet> cloudlets, List<Vm> vms) {
    for (int i = 0; i < cloudlets.size(); i++) {
        Cloudlet cloudlet = cloudlets.get(i);
        Vm vm = vms.get(i);
        cloudlet.setVmId(vm.getId());
    }
}


    private static void printCloudletList(List<Cloudlet> list) {
        int size = list.size();
        Cloudlet cloudlet;

        String indent = "    ";
        Log.printLine();
        Log.printLine("========== OUTPUT ==========");
        Log.printLine("Cloudlet ID" + indent + "STATUS" + indent
                + "Data center ID" + indent + "VM ID" + indent + "Time" + indent
                + "Start Time" + indent + "Finish Time");

        DecimalFormat dft = new DecimalFormat("###.##");
        for (int i = 0; i < size; i++) {
            cloudlet = list.get(i);
            Log.print(indent + cloudlet.getCloudletId() + indent + indent);

            if (cloudlet.getCloudletStatus() == Cloudlet.SUCCESS) {
                Log.print("SUCCESS");

                Log.printLine(indent + indent + cloudlet.getResourceId()
                        + indent + indent + indent + cloudlet.getVmId()
                        + indent + indent
                        + dft.format(cloudlet.getActualCPUTime()) + indent
                        + indent + dft.format(cloudlet.getExecStartTime())
                        + indent + indent
                        + dft.format(cloudlet.getFinishTime()));
            }
        }
    }
}

```