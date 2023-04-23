Download Link: https://assignmentchef.com/product/solved-accelerators-and-accelerated-systems-homework-assignment-3
<br>
Make sure to submit a zip/tar.gz archive with all necessary files: ex3.cu, ex2.cu and additional header files (.h) you created if you have done so. Archive name should be IDs of the students, separated with underscore

(e.g. 123456789_123571113.tar.gz)




In this assignment, we will implement an RDMA-based client application performing the algorithm from homework 1 using the producer-consumer queue you developed in homework 2.




<strong>Please make sure that in all versions of your GPU implementation the average distance your implementation produces is equal to what the CPU implementation produces. </strong>




<h1>Homework package</h1>

The archive includes the following files:

<table width="642">

 <tbody>

  <tr>

   <td width="321"><strong>Filename </strong></td>

   <td width="321"><strong>Description </strong></td>

  </tr>

  <tr>

   <td width="321">homework3.pdf</td>

   <td width="321">This file.</td>

  </tr>

  <tr>

   <td width="321">ex3.cu</td>

   <td width="321">A template for you to implement the exercise and submit. <strong>You may edit this file.</strong></td>

  </tr>

  <tr>

   <td width="321">ex2.cu</td>

   <td width="321">Example solution of the second part of homework 2. <strong>You may edit this file.</strong></td>

  </tr>

  <tr>

   <td width="321">ex3-cpu.cu</td>

   <td width="321">A CPU implementation of the algorithm that is used to check the results.</td>

  </tr>

  <tr>

   <td width="321">server.cu</td>

   <td width="321">An RDMA server that processes images on the GPU (using your code in ex2/3.cu).</td>

  </tr>

  <tr>

   <td width="321">client.cu</td>

   <td width="321">An RDMA client that sends images to be processed to the server and verifies its output (using your code in ex3.cu).</td>

  </tr>

  <tr>

   <td width="321">common.cu</td>

   <td width="321">Common RDMA code used by both the client and the server.</td>

  </tr>

  <tr>

   <td width="321">Makefile</td>

   <td width="321">Allows building the client and the server using make. This makefile adds the maxrregcount=32 argument to the nvcc command in order to control the amount of registers used and the -arch=sm_75 to compile for Turing GPUs and support C++ atomics.</td>

  </tr>

  <tr>

   <td width="321">ex2.h</td>

   <td width="321">Header file with declarations of ex2.cu functions and structs needed by client.cu/server.cu/ex2.cu/ex3.cu.</td>

  </tr>

  <tr>

   <td width="321">ex3.h</td>

   <td width="321">Header file with declarations of ex3.cu functions and structs needed by client.cu/server.cu/ex2.cu/ex3.cu.</td>

  </tr>

 </tbody>

</table>




<h1>Submission Instructions</h1>

You will be submitting an archive (id1_id2.tar.gz or id1_id2.zip) with at least 2 files:

<ol>

 <li>cu, ex2.cu, and additional files you have created:

  <ul>

   <li>The cu contains the client and the server classes you will implement.</li>

   <li>The cu file contains the GPU queues code. You may use the file provided in the homework package or modify it.</li>

   <li>If you need to, create additional .h header files to allow shared declarations between cu and ex3.cu. Note that due to a bug in CUDA 10.2, only a single object file may include &lt;cuda/atomic&gt;. As a workaround, ex3.cu has been updated to include ex2.cu, instead of compiling them separately.</li>

   <li>Do not modify other files in the package.</li>

   <li>Make sure to check for errors for all RDMA verbs calls and CUDA calls.</li>

   <li>Make sure your code compiles <strong>without warnings</strong> and runs correctly before submitting.</li>

   <li>Free all memory and release resources once you have finished using them.</li>

  </ul></li>

</ol>




<h1>Neighborliness</h1>

Machines in this course are being shared among several student teams. Please be sure to clean up after yourselves when you are done with a machine:

<ol>

 <li>GPU resources, RDMA resources, and TCP sockets are released when the process terminates. Do not leave processes running or stopped.</li>

 <li>Do not leave open debugging sessions such as nsight or cuda-gdb with running or stopped processes.</li>

</ol>

<h1>General Description of the problem</h1>

In this assignment, we will implement a network client-server version of the image quantization server from homework 2 using RDMA Verbs. We will control the CPU-GPU queues remotely using RDMA from the client.

As an example, you are provided with an RPC-based implementation. Your task will be to implement a second version in which the client uses RDMA to control the GPU.




RPC Protocol

This version is given as an example you can use to learn how to implement the second part. <strong>No need to implement this part yourselves</strong>.

The main RPC protocol is:




<ol>

 <li>The client sends a request to the server using a <strong>Send</strong> Each request contains a unique identifier chosen by the client, and the parameters the server needs to access the input and output images remotely (Key, Address).</li>

 <li>The server performs <strong>RDMA Read</strong> operation to read the input image from the address specified by the client.</li>

 <li>The server performs its task (image quantization) using the GPU.</li>

 <li>The server uses an <strong>RDMA Write with Immediate </strong>operation to send the output image back to the client at the requested address. The immediate value is the unique identifier chosen by the client for this request in step 1.</li>

 <li>The client receives a completion, notices the task has completed and continues to the next task.</li>

</ol>

We use a special RPC request to indicate that the server needs to terminate. Such a request causes the server to terminate without doing steps 2-3, replying with an empty RDMA Write with Immediate operation (step 4) to let the client know it is terminating.




You can find the server code for this protocol in the server_rpc_context class and the client code in the client_rpc_context class.

<em>Figure 1 RPC protocol illustration</em>




<em>RPC protocol connection establishment </em>

The client and server need to establish an InfiniBand connection and to exchange parameters for the above protocol to work. They use the following steps to do that.

<ol>

 <li>Both client and server allocate required RDMA resources, such as PD, QP, CQ, etc (in the rdma_context::initialize_verbs method).</li>

 <li>Both allocate an array of requests (part of the rdma_context parent class), to allow the client to send requests to the server over an RDMA QP. The initialize_verbs method also registers these arrays with the NIC’s memory system and creates an MR. 3. The client class receives pointers to the input and output buffers to register them as MRs as well (set_input_images/set_output_images). These buffers are registered with RDMA access, allowing the server to read/write them remotely.</li>

 <li>Server and client briefly use a TCP socket (rdma_context::tcp_connection) to exchange information about their RDMA address (LID/GID/IP address, QP number), as well as information about the RDMA buffers (Key, Address), using the</li>

</ol>

send_connection_establishment_data and recv_connection_establishment_data.

<ol start="5">

 <li>Now we can do real work over the established RDMA connection. We start performing RPC calls as described above. The server does that in its server_rpc_context::event_loop method, while the client uses its</li>

</ol>

client_rpc_context::enqueue and client_rpc_context::dequeue methods from a main loop in client.cu.




Client-side RDMA protocol

An alternative protocol we will examine through this assignment uses your CPU-GPU producer-consumer queues from homework 2 remotely from the client using RDMA. You may use your GPU-side code as is. However, rather than having the server process write tasks directly to the GPU queues and poll the GPU queues, the client process will do that through RDMA.

The server tasks are:

<ol>

 <li>Initialize CUDA environment, allocate CPU buffers mapped for the GPU, and instantiate the CUDA kernel (taken from homework 2).</li>

 <li>Register the memory that the GPU accesses also for remote access through verbs.</li>

 <li>Establish connection with the client and send the client all necessary information to access these buffers over TCP, as well as other information needed by the client (e.g., the number of GPU queues).</li>

</ol>

After connection is established, the server CPU is not involved. It only waits for an RPC message to indicate termination.

The client enqueue and dequeue operations are like those used in homework 2, but here they use RDMA to access the GPU queues remotely. To enqueue, the client:

<ol>

 <li>Checks if there is room on the GPU queues to submit new tasks, using an RDMA read operation.</li>

 <li>If there is room, copy the image to the server and enqueue the next task on the</li>

</ol>

GPU’s queue using RDMA write operations.

For dequeuing, the client:

<ol>

 <li>Polls the GPU queues using RDMA read operations for any tasks that have completed. You may wait for the RDMA read operation to complete, but you must not wait for the GPU to complete each task before moving on to the next step.</li>

 <li>If a task has complete, use RDMA read operations to read the image ID and image content back to the client machine.</li>

</ol>

<h1>Setup</h1>

We’ll be running both the server and the client on the same machine. The RDMA card in the machine (ConnectX-4 Lx) has two physical ports, connected to each other via an Ethernet cable. Each port has a separate network device. The server and the client will both use port 1 by default, communicating via NIC loopback.




<h1>Building and running</h1>

Compile both the server and the client using the make command.




Both the server and the client accept two command line arguments:

<ol>

 <li>Type of the run: <strong>rpc</strong> or <strong>queue</strong>.</li>

 <li>TCP port to use. If the server’s TCP port argument is omitted, a random port is chosen. Run the server, and after it starts run the client with the same port and mode of operation.</li>

</ol>




<h1>Your tasks</h1>

The client and server already implement the RPC protocol. Your task is to implement the queue version. The missing parts are marked with TODO comments with instructions of what must be done. Specifically, you would need to perform the following steps. For the server, implement the changes in the server_queues_context class:

<ol>

 <li>In the constructor:

  <ol>

   <li>register the host memory accessible by the GPU through RDMA verbs, to create one or more memory regions that are accessible remotely.</li>

   <li>Exchange the parameters of the GPU queues (Memory region keys, addresses, number of queues, etc.) with the client. You may use the send_over_socket and recv_over_socket methods for that.</li>

   <li>Start the persistent kernel on the GPU.</li>

  </ol></li>

 <li>In the destructor:

  <ol>

   <li>Make sure the GPU kernel terminates and teardown the added memory regions.</li>

  </ol></li>

 <li>In the event_loop method:

  <ol>

   <li>There is no need to handle messages here. Just wait for termination message from the client, return a reply and exit the method.</li>

  </ol></li>

</ol>




The client should be implemented as part of the client_queues_context class:

<ol>

 <li>Add necessary state to track the CPU-GPU queues on the client side to the class as member variables (e.g. client’s / CPU’s producer indices and consumer indices), as well as information about the server such as number of queues, rkeys and addresses.</li>

 <li>Add memory regions the client may use (e.g. for RDMA operations toward the server).</li>

 <li>In the constructor:

  <ol>

   <li>Communicate with the server to receive the necessary parameters.</li>

   <li>Register memory regions defined in (2).</li>

  </ol></li>

 <li>In the set_input_images/set_output_images register memory regions for the client’s input and output buffers. Pay attention to the necessary access permissions.</li>

 <li>In the destructor:

  <ol>

   <li>Send a termination message to the server and wait for a response.</li>

   <li>Release registered memory regions.</li>

  </ol></li>

 <li>Implement the enqueue method:

  <ol>

   <li>Use RDMA Read operations to check whether the destination queue is full.</li>

   <li>Return false for a full queue.</li>

   <li>Use RDMA Write operations to write the data to the queue.</li>

   <li>Use RDMA Write to increment the producer index.</li>

  </ol></li>

 <li>Implement the dequeue method:

  <ol>

   <li>Use RDMA Read operations to check whether the desired queue is empty. i. Return false if so.</li>

   <li>Use RDMA Read operations to read the data from the queue.</li>

   <li>Use RDMA Write to increment the consumer index.</li>

  </ol></li>

</ol>




You’ll be using these verbs: ibv_post_send(), ibv_reg_mr(), ibv_poll_cq(). You can see their usage in the provided code in the RPC server example, in the man pages (man ibv_post_send), or in the RDMAmojo website, which has excellent explanations and examples for these verbs: <a href="http://www.rdmamojo.com/2013/01/26/ibv_post_send/">http://www.rdmamojo.com/2013/01/26/ibv_post_send/</a> <a href="http://www.rdmamojo.com/2013/02/15/ibv_poll_cq/">http://www.rdmamojo.com/2013/02/15/ibv_poll_cq/</a> <a href="http://www.rdmamojo.com/2012/09/07/ibv_reg_mr/">http://www.rdmamojo.com/2012/09/07/ibv_reg_mr/</a>