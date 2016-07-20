//Cache Coherence Protocol

//Cache_A
module Cache_A(

//Global inputs and outputs 
clock,

//Asynchronous active low reset
 rst_l,

//Inputs from CPU 
cpu_cac_add_A, 
cpu_cac_rd_A, 
cpu_cac_wrt_A, 
cpu_cac_data_A,

//Inputs from Main Memory 
mem_cac_data_A, 
data_avail_memA,

//Inputs from mmu 
//mmu_cac_add_A,

//Inputs to the Memory Bus Controller 
bus_bus_bus_req_A, 
priority_bus_bus_req_A,

//Snooping input ports 
snoop_B, 
snoop_add_B, 
invalidate_B,


//Outputs to CPU 
cac_cpu_hit_A, 
cac_cpu_miss_A, 
cac_cpu_data_A,

//Outputs to Main Memory 
cac_data_avail_mem_Add_A, 
cac_mem_data_A, 
cac_mem_rd_A, 
cac_mem_wrt_A, 
priority_wrt_A,

//Outputs from Memory Bus Controller 
bus_A,

//Snooping output ports 
snoop_A, 
snoop_add_A, 
invalidate_A

);

// Input Ports
 



//Global 
input clock; input rst_l;

//CPU

input [4:0]cpu_cac_data_A; 
input [4:0]cpu_cac_add_A; 
input cpu_cac_rd_A;
input cpu_cac_wrt_A;

// Memory

input [4:0]mem_cac_data_A; 
input data_avail_memA;

//Memory Bus Controller 
input bus_A;

//Snooping 
input snoop_B;
input [4:0] snoop_add_B; 
input invalidate_B;

// Output Ports

//CPU
output [4:0] cac_cpu_data_A; 
output cac_cpu_hit_A; 
output cac_cpu_miss_A;

// Memory
output [4:0] cac_mem_data_A;
output [4:0] cac_data_avail_mem_Add_A; 
output cac_mem_rd_A;
output cac_mem_wrt_A; 
output priority_wrt_A;

//Memory Bus Controller 
output bus_bus_bus_req_A; 
output priority_bus_bus_req_A;

//Snooping 
output snoop_A;
output [4:0] snoop_add_A; 
output invalidate_A;

// Registers

//CPU

reg [4:0] cac_cpu_data_A; 
reg cac_cpu_hit_A;
reg cac_cpu_miss_A;

// Memory

reg [4:0] cac_mem_data_A; 
reg [4:0] cac_memAdd_A; 
reg cac_mem_rd_A;
reg cac_mem_wrt_A; 
reg priority_wrt_A;

//Memory Bus Controller 
reg bus_bus_bus_req_A; 
reg priority_bus_bus_req_A;

//Snooping 
reg snoop_A;
reg [4:0] snoop_add_A; 
reg snoop_page_1;
reg snoop_page_2; 
reg invalidate_A;

//Cache Buffer
reg [10:0] buffer_1 [0:7]; 
reg [10:0] buffer_2 [0:7];

//Read/Write regs 
reg read_A;
reg write_A; 
reg dirty;

//Cache FSM 
reg [2:0] state; 
reg [2:0] next_s;
reg [2:0] back_t_s;

// Net

//Cache-CPU Interface
wire [10:0] cpu_buffer_1; //Buffer value at current index in page 0
wire [10:0] cpu_buffer_2; //Buffer value at current index in page 1
wire [2:0] cpu_index;  //Index value from CPU addess
wire [1:0] cpu_tag;    //Tag value from CPU addess
wire [1:0] cur_tag_1;
wire [1:0] cur_tag_2;
wire [4:0] add_mem;
wire [4:0] cac_data;   //Current Cache Data
wire [4:0] mem_data;
wire [1:0] snoop_tag;
wire [2:0] snoop_index; 
wire [10:0] snoop_buffer_1; 
wire [10:0] snoop_buffer_2; 
wire [10:0] cac_data_1; 
wire [10:0] cac_data_2; 
wire valid_1;
wire valid_2; 
wire dirty_1; 
wire dirty_2; 
wire update_1; 
wire update_2;

//Integer
integer i;

//Parameters
parameter S0 = 0;	// Initial
parameter S1 = 1;	// Wait State 1
parameter S2 = 2;	// Store State
parameter S3 = 3;	// Wait State 2
parameter S4 = 4;
parameter S5 = 5;	//priority_write state

//Assign

//Cache-CPU
assign cpu_tag = cpu_cac_add_A[1:0]; 
assign cur_tag_1 = cpu_buffer_1[6:5]; 
assign cur_tag_2 = cpu_buffer_2[6:5]; 
assign cpu_index = cpu_cac_add_A[4:2];
assign add_mem = cpu_cac_add_A[4:0];
assign valid_1 = cpu_buffer_1[10]; 
assign valid_2 = cpu_buffer_2[10]; 
assign dirty_1 = cpu_buffer_1[9]; 
assign dirty_2 = cpu_buffer_2[9];
assign update_1 = cpu_buffer_1[7]; 
assign update_2 = cpu_buffer_2[7];
assign cpu_buffer_1 = buffer_1[cpu_index]; 
assign cpu_buffer_2 = buffer_2[cpu_index];

//Cache- Memory
assign cac_data_1 = buffer_1[cpu_index]; 
assign cac_data_2 = buffer_2[cpu_index];
assign mem_data = mem_cac_data_A[4:0];
assign snoop_tag = snoop_add_B[1:0]; 
assign snoop_index = snoop_add_B[4:2];
assign snoop_buffer_1 = buffer_1[snoop_index]; 
assign snoop_buffer_2 = buffer_2[snoop_index];

always @(negedge clock) 
next_s <= state;

always @(posedge clock or negedge rst_l) 
begin
if(rst_l == 0) 
begin
cac_mem_rd_A <= 1'b0; 
cac_mem_wrt_A <= 1'b0; 
cac_cpu_data_A <= 5'b0; 
cac_cpu_miss_A <= 1'b0; 
cac_cpu_hit_A <= 1'b0; 
cac_mem_data_A <= 5'b0; 
cac_memAdd_A <= 5'b0; 
snoop_A <= 1'b0; 
invalidate_A <= 1'b0; 
bus_bus_bus_req_A <= 1'b0;
state <= S0;

for(i=0; i <= 7; i = i+1)
begin
buffer_1[i] <= 11'b0;
buffer_2[i] <= 11'b0;
end
end
else	
begin 
case(next_s)
S0:
begin
cac_cpu_data_A <= 5'b0; 
cac_cpu_miss_A <= 1'b0; 
cac_cpu_hit_A <= 1'b0; 
cac_mem_data_A <= 5'b0; 
cac_memAdd_A <= 5'b0; 
cac_mem_rd_A <= 1'b0; 
cac_mem_wrt_A <= 1'b0; 
priority_bus_bus_req_A <= 1'b0; 
snoop_A <= 1'b0; 
snoop_page_1 <= 1'b0;
snoop_page_2 <= 1'b0; 
invalidate_A <= 1'b0;
read_A <= 1'b0; 
write_A <= 1'b0; 
dirty <= 1'b0;
priority_wrt_A <= 1'b0; 
bus_bus_bus_req_A <= 1'b0;

//If it is a read
if(cpu_cac_rd_A == 1'b1) 
begin
read_A <= 1'b1;
if(cpu_tag == cur_tag_1) 
begin
if(valid_1 == 1'b0)
begin
$display ("Read Miss In Invalid State PAGE 1"); 
cac_cpu_miss_A <= 1'b1;
bus_bus_bus_req_A <= 1'b1; 
state <= S1;
end

//read hit in EXCLUSIVE state
if((dirty_1 == 1'b1) & (valid_1 == 1'b1)) 
begin
$display ("Read Hit In Exculsive State PAGE 1"); 
cac_cpu_hit_A <= 1'b1;
cac_cpu_data_A <= cpu_buffer_1[4:0];
buffer_1[cpu_index] <= buffer_1[cpu_index] | 11'b00010000000; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111; 
state <= S0;
end

//read hit in SHARED state
if((dirty_1 == 1'b0) & (valid_1 == 1'b1)) 
begin
$display ("Read Hit In Shared State PAGE 1"); 
cac_cpu_hit_A <= 1'b1;
cac_cpu_data_A <= cpu_buffer_1[4:0];
buffer_1[cpu_index] <= buffer_1[cpu_index] | 11'b00010000000; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111;
state <= S0; 
end
end
if(cpu_tag == cur_tag_2) //Check Page 2 of the Cache 
begin
if(valid_2 == 1'b0) 
begin
$display ("Read Miss In Invalid State PAGE 2"); 
cac_cpu_miss_A <= 1'b1;
bus_bus_bus_req_A <= 1'b1; 
state <= S1;
end

//read hit in EXCLUSIVE state 
if((dirty_2 == 1'b1) & (valid_2 == 1'b1)) 
begin
$display ("Read Hit In Exculsive State PAGE 2"); 
cac_cpu_hit_A <= 1'b1;
cac_cpu_data_A <= cpu_buffer_2[4:0];
buffer_2[cpu_index] <= buffer_2[cpu_index] | 11'b00010000000; 
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111;
state <= S0; 
end
if((dirty_2 == 1'b0) & (valid_2 == 1'b1)) 
begin
$display ("Read Hit In Shared State PAGE 2"); 
cac_cpu_hit_A <= 1'b1;
cac_cpu_data_A <= cpu_buffer_2[4:0];
buffer_2[cpu_index] <= buffer_2[cpu_index] | 11'b00010000000; 
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111; 
state <= S0;
end
end
if((cpu_tag != cur_tag_1) & (cpu_tag != cur_tag_2)) 
begin
$display ("Read Miss In Invalid State"); 
cac_cpu_miss_A <= 1'b1;
bus_bus_bus_req_A <= 1'b1;
state <= S1; 
end
end
if(cpu_cac_wrt_A == 1'b1) 
begin
write_A <= 1'b1; 
if(cpu_tag == cur_tag_1) 
begin

//write miss in INVALID state 
if(valid_1 == 1'b0) 
begin
$display ("Write Miss In Invalid State PAGE 1"); 
cac_cpu_miss_A <= 1'b1;
bus_bus_bus_req_A <= 1'b1; 
state <= S1;
end

//write miss in EXCLUSIVE state
else if((dirty_1 == 1'b1) & (valid_1 == 1'b1)) 
begin
$display ("Write Hit In Exculsive State PAGE 1"); 
cac_cpu_hit_A <= 1'b1;
buffer_1[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_A}; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111; 
state <= S0;
end

//write hit in SHARED state
else if((dirty_1 == 1'b0) & (valid_1 == 1'b1)) 
begin
$display ("Write Hit In Shared State PAGE 1");
buffer_1[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_A}; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111; 
bus_bus_bus_req_A <= 1'b1;
back_t_s <= S0; 
state <= S4;
end
end
else if(cpu_tag == cur_tag_2) 
begin
if(valid_2 == 1'b0) 
begin
$display ("Write Miss In Invalid State PAGE 2"); 
cac_cpu_miss_A <= 1'b1;
bus_bus_bus_req_A <= 1'b1; 
state <= S1;
end

//write hit in EXCLUSIVE state
else if((dirty_2 == 1'b1) & (valid_2 == 1'b1)) 
begin
$display ("Write Hit In Exculsive State PAGE 2"); 
cac_cpu_hit_A <= 1'b1;
buffer_2[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_A};
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111;
state <= S0;
end

//write hit in SHARED state
else if((dirty_2 == 1'b0) & (valid_2 == 1'b1)) 
begin
$display ("Write Hit In Shared State PAGE 2");
buffer_2[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_A};
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111; 
bus_bus_bus_req_A <= 1'b1;
back_t_s <= S0; 
state <= S4;
end
end
else if((cpu_tag != cur_tag_1) & (cpu_tag != cur_tag_2)) 
begin

$display ("Write Miss In Invalid state"); cac_cpu_miss_A <= 1'b1; bus_bus_bus_req_A <= 1'b1;
state <= S1;
end
end

//if snooping
 
if(snoop_B == 1'b1) begin

if((snoop_tag == snoop_buffer_1[6:5])&(snoop_buffer_1[10] == 1'b1)) begin

if(snoop_buffer_1[9] == 1'b1) begin

priority_bus_bus_req_A <= 1'b1; snoop_page_1 <= 1'b1; back_t_s <= S0;

state <= S3; end

else begin

state <= S0; end

end

else if((snoop_tag == snoop_buffer_2[6:5])&(snoop_buffer_2[10] == 1'b1)) begin

if(snoop_buffer_2[9] == 1'b1) begin

priority_bus_bus_req_A <= 1'b1; snoop_page_2 <= 1'b1; back_t_s <= S0;

state <= S3; end

else begin

state <= S0; end
end

else begin

state <= S0; end
end

//if Invalidation

if(invalidate_B == 1'b1) begin

if((snoop_tag == snoop_buffer_1[6:5])&(snoop_buffer_1[10] == 1'b1)) begin

buffer_1[snoop_index] <= (buffer_1[snoop_index] & 11'b00101111111); buffer_2[snoop_index] <= (buffer_2[snoop_index] | 11'b00010000000);

state <= S0; end

else if((snoop_tag == snoop_buffer_2[6:5])&(snoop_buffer_1[10] == 1'b1)) begin

buffer_2[snoop_index] <= (buffer_2[snoop_index] & 11'b00101111111); buffer_1[snoop_index] <= (buffer_1[snoop_index] | 11'b00010000000);
state <= S0;

end else begin

state <= S0; end

end
end //State 0

//State S0 - Wait

S1:

begin

cac_cpu_miss_A <= 1'b0; priority_wrt_A <= 1'b0; 
$display ("Waiting in S1"); 
if(bus_A == 1'b1)

begin

cac_mem_rd_A <= 1'b1; cac_memAdd_A <= add_mem; 
snoop_A <= 1'b1; snoop_add_A <= add_mem; bus_bus_bus_req_A <= 1'b0; state <= S2;

end

//if invalidation

else if(invalidate_B == 1'b1) begin

if((snoop_tag == snoop_buffer_1[6:5])&(snoop_buffer_1[10] == 1'b1)) begin

buffer_1[snoop_index] <= (buffer_1[snoop_index] & 11'b00101111111);
 buffer_2[snoop_index] <= (buffer_2[snoop_index] | 11'b00010000000);

if(snoop_add_B == cpu_cac_add_A) begin

bus_bus_bus_req_A <= 1'b0; cac_cpu_miss_A <= 1'b1; state <= S1;

end else begin

state <= S1; end

end

else if((snoop_tag == snoop_buffer_2[6:5])&(snoop_buffer_2[10] == 1'b1)) begin

buffer_2[snoop_index] <= (buffer_2[snoop_index] & 11'b00101111111);
 buffer_1[snoop_index] <= (buffer_1[snoop_index] | 11'b00010000000);

if(snoop_add_B == cpu_cac_add_A) begin

bus_bus_bus_req_A <= 1'b0; cac_cpu_miss_A <= 1'b1; state <= S1;

end else begin

state <= S1; end
end

else
begin

state <= S1; end

end else begin

state <= S1; end
end //S1

//State S2
 
S2: begin

dirty <= 1'b0; snoop_A <= 1'b0;

cac_mem_wrt_A <= 1'b0; cac_mem_rd_A <= 1'b0; snoop_page_1 <= 1'b0; snoop_page_2 <= 1'b0; priority_wrt_A <= 1'b0;

if((data_avail_memA == 1'b1)|(dirty == 1'b1)) begin

if(update_1 == 1'b0) 
 begin

if(dirty_1 == 1'b1) begin

cac_mem_wrt_A <= 1'b1; cac_memAdd_A <= add_mem;
cac_mem_data_A <= cpu_cac_data_A;

buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b10111111111; dirty <= 1'b1;

state <= S2; end

else if(dirty_1 == 1'b0) begin

buffer_1[cpu_index] <= {1'b1,1'b0,1'b0,1'b1,cpu_tag,mem_cac_data_A}; buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111; if((read_A == 1'b1) & (write_A == 1'b0))
begin

state <= S0; end

else if((read_A == 1'b0) & (write_A == 1'b1)) begin

buffer_1[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_A}; cac_mem_wrt_A <= 1'b1;

cac_memAdd_A <= add_mem; cac_mem_data_A <= cpu_cac_data_A; invalidate_A <= 1'b1;

snoop_add_A <= add_mem; state <= S0;

end

end end

else if(update_2 == 1'b0) begin

if(dirty_2 == 1'b1) begin

cac_mem_wrt_A <= 1'b1; cac_memAdd_A <= add_mem; cac_mem_data_A <= cpu_cac_data_A;

buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b10111111111; dirty <= 1'b1;

state <= S2;
end

else if(dirty_2 == 1'b0) begin

buffer_2[cpu_index] <= {1'b1,1'b0,1'b0,1'b1,cpu_tag,mem_cac_data_A}; buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111; if((read_A == 1'b1) & (write_A == 1'b0))
begin

state <= S0; end
 
if((read_A == 1'b0) & (write_A == 1'b1)) begin

buffer_2[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_A}; cac_mem_wrt_A <= 1'b1;

cac_memAdd_A <= add_mem; cac_mem_data_A <= cpu_cac_data_A; invalidate_A <= 1'b1;

snoop_add_A <= add_mem; state <= S0;
end

end end
end

//if snooping

else if(snoop_B == 1'b1) begin

if((snoop_tag == snoop_buffer_1[6:5]) & (snoop_buffer_1[10] == 1'b1)) begin

if(snoop_buffer_1[9] == 1'b1) begin

priority_bus_bus_req_A <= 1'b1; snoop_page_1 <= 1'b1; back_t_s <= S2;

state <= S3; end

else begin

state <= S2; end
end

else if((snoop_tag == snoop_buffer_2[6:5]) & (snoop_buffer_2[10] == 1'b1)) begin

if(snoop_buffer_2[9] == 1'b1) begin

priority_bus_bus_req_A <= 1'b1; snoop_page_2 <= 1'b1; back_t_s <= S2;

state <= S3; end

else begin

state <= S2; end

end

else begin

state <= S2; end

end

//if invalidation

else if(invalidate_B == 1'b1) begin

if((snoop_tag == snoop_buffer_1[6:5])& (snoop_buffer_1[10] == 1'b1)) begin

buffer_1[snoop_index] <= buffer_1[snoop_index] & 11'b00110111111; buffer_2[snoop_index] <= buffer_2[snoop_index] | 11'b00001000000; state <= S2;

end

else if((snoop_tag == snoop_buffer_2[6:5]) & (snoop_buffer_2[10] == 1'b1)) begin

buffer_2[snoop_index] <= buffer_2[snoop_index] & 11'b00110111111; buffer_1[snoop_index] <= buffer_1[snoop_index] | 11'b00001000000;

state <= S2;
end

else begin

state <= S2; end

end end // S2


// State S3

S3: begin

if(bus_A == 1'b1) begin
priority_bus_bus_req_A <= 1'b0;

if((snoop_page_1 == 1'b1)&(snoop_page_2 == 1'b0)) begin

priority_wrt_A <= 1'b1; cac_memAdd_A <= snoop_add_B;
cac_mem_data_A <= snoop_buffer_1[4:0];

buffer_1[snoop_index] <= buffer_1[snoop_index] & 11'b10111111111; state <= S5;
end

else if((snoop_page_1 == 1'b0)&(snoop_page_2 == 1'b1)) begin

priority_wrt_A <= 1'b1; cac_memAdd_A <= snoop_add_B;
cac_mem_data_A <= snoop_buffer_2[4:0];

buffer_2[snoop_index] <= buffer_2[snoop_index] & 11'b10111111111; state <= S5;
end
end

else
begin

state <= S3; end

end//S3

//State S4

S4: begin

if(bus_A == 1'b1) begin

bus_bus_bus_req_A <= 1'b0; 
cac_cpu_hit_A <= 1'b1;

invalidate_A <= 1'b1; snoop_add_A <= add_mem; cac_mem_wrt_A <= 1'b1;
  

cac_memAdd_A <= add_mem; cac_mem_data_A <= cpu_cac_data_A; state <= back_t_s;

end
end // S4

//State S5

S5: begin

state <= back_t_s; end

endcase end
end

endmodule //Cache_A

// Cache_B

module Cache_B(

//Global inputs and outputs 
clock,

//Asynchronous active low reset
 rst_l,

//Inputs from CPU 
cpu_cac_add_B, cpu_cac_rd_B, cpu_cac_wrt_B, cpu_cac_data_B,

//Inputs from Main Memory 
mem_cac_data_B, data_avail_memB,

//Inputs from mmu 
//mmu_cac_add_B,

//Inputs to the Memory Bus Controller 
bus_bus_bus_req_B, priority_bus_bus_req_B,

//Snooping input ports 
snoop_A, snoop_add_A, invalidate_A,


//Outputs to CPU 
cac_cpu_hit_B, cac_cpu_miss_B, cac_cpu_data_B,

//Outputs to Main Memory 
cac_data_avail_mem_Add_B, cac_mem_data_B, cac_mem_rd_B, cac_mem_wrt_B, priority_wrt_B,

//Outputs from Memory Bus Controller 
bus_B,

//Snooping output ports 
snoop_B, snoop_add_B, invalidate_B

);

// Input Ports
 
//Global 
input clock; 
input rst_l;

//CPU

input [4:0]cpu_cac_data_B; 
input [4:0]cpu_cac_add_B; 
input cpu_cac_rd_B;
input cpu_cac_wrt_B;

// Memory

input [4:0]mem_cac_data_B; 
input data_avail_memB;

//Memory Bus Controller 
input bus_B;

//Snooping 
input snoop_A;
input [4:0] snoop_add_A; 
input invalidate_A;

// Output Ports

//CPU

output [4:0] cac_cpu_data_B; 
output cac_cpu_hit_B; 
output cac_cpu_miss_B;

// Memory
output [4:0] cac_mem_data_B;
output [4:0] cac_data_avail_mem_Add_B; 
output cac_mem_rd_B;
output cac_mem_wrt_B;
output priority_wrt_B;

//Memory Bus Controller 
output bus_bus_bus_req_B; 
output priority_bus_bus_req_B;

//Snooping 
output snoop_B;
output [4:0] snoop_add_B; 
output invalidate_B;

// Registers

//CPU

reg [4:0] cac_cpu_data_B; 
reg cac_cpu_hit_B;
reg cac_cpu_miss_B;

// Memory

reg [4:0] cac_mem_data_B; 
reg [4:0] cac_memAdd_B; 
reg cac_mem_rd_B;
reg cac_mem_wrt_B; 
reg priority_wrt_B;

//Memory Bus Controller 
reg bus_bus_bus_req_B; 
reg priority_bus_bus_req_B; 
 
//Snooping 
reg snoop_B;
reg [4:0] snoop_add_B; 
reg snoop_page_1;
reg snoop_page_2; 
reg invalidate_B;

//Cache Buffer

reg [10:0] buffer_1 [0:7]; 
reg [10:0] buffer_2 [0:7];

//Read/Write regs 
reg read_B;
reg write_B; 
reg dirty;

//Cache FSM 
reg [2:0] state; 
reg [2:0] next_s;
reg [2:0] back_t_s;

// Net
//Cache-CPU Interface

wire [10:0] cpu_buffer_1; //Buffer value at current index in page 0
wire [10:0] cpu_buffer_2; //Buffer value at current index in page 1
wire [2:0] cpu_index;  //Index value from CPU addess
wire [1:0] cpu_tag;    //Tag value from CPU addess
wire [1:0] cur_tag_1;
wire [1:0] cur_tag_2;
wire [4:0] add_mem;
wire [4:0] cac_data;   //Current Cache Data
wire [4:0] mem_data;
wire [1:0] snoop_tag;
wire [2:0] snoop_index; 
wire [10:0] snoop_buffer_1; 
wire [10:0] snoop_buffer_2;
wire [10:0] cac_data_1; 
wire [10:0] cac_data_2; 
wire valid_1;
wire valid_2;
wire dirty_1; 
wire dirty_2; 
wire update_1; 
wire update_2;

//Integer

integer i;

//Parameters

parameter S0 = 0;	// Initial
parameter S1 = 1;	// Wait State 1
parameter S2 = 2;	// Store State
parameter S3 = 3;	// Wait State 2
parameter S4 = 4;
parameter S5 = 5;	//priority_write state

//Assign
//Cache-CPU

assign cpu_tag = cpu_cac_add_B[1:0]; 
assign cur_tag_1 = cpu_buffer_1[6:5]; 
assign cur_tag_2 = cpu_buffer_2[6:5];
assign cpu_index = cpu_cac_add_B[4:2];
assign add_mem = cpu_cac_add_B[4:0];
assign valid_1 = cpu_buffer_1[10]; 
assign valid_2 = cpu_buffer_2[10]; 
assign dirty_1 = cpu_buffer_1[9]; 
assign dirty_2 = cpu_buffer_2[9];
assign update_1 = cpu_buffer_1[7]; 
assign update_2 = cpu_buffer_2[7];
assign cpu_buffer_1 = buffer_1[cpu_index]; 
assign cpu_buffer_2 = buffer_2[cpu_index];


//Cache- Memory

assign cac_data_1 = buffer_1[cpu_index]; 
assign cac_data_2 = buffer_2[cpu_index];
assign mem_data = mem_cac_data_B[4:0];
assign snoop_tag = snoop_add_A[1:0]; 
assign snoop_index = snoop_add_A[4:2];
assign snoop_buffer_1 = buffer_1[snoop_index]; 
assign snoop_buffer_2 = buffer_2[snoop_index];

always @(negedge clock) 
next_s <= state;
always @(posedge clock or negedge rst_l) 
begin
if(rst_l == 0) 
begin

cac_mem_rd_B <= 1'b0; 
cac_mem_wrt_B <= 1'b0; 
cac_cpu_data_B <= 5'b0; 
cac_cpu_miss_B <= 1'b0; 
cac_cpu_hit_B <= 1'b0; 
cac_mem_data_B <= 5'b0; 
cac_memAdd_B <= 5'b0; 
snoop_B <= 1'b0; 
invalidate_B <= 1'b0; 
bus_bus_bus_req_B <= 1'b0;
state <= S0;

for(i=0; i <= 7; i = i+1)
begin
buffer_1[i] <= 11'b0;
buffer_2[i] <= 11'b0;
end
end
else	
begin 
case(next_s)

S0:
begin
cac_cpu_data_B <= 5'b0; 
cac_cpu_miss_B <= 1'b0; 
cac_cpu_hit_B <= 1'b0; 
cac_mem_data_B <= 5'b0; 
cac_memAdd_B <= 5'b0; 
cac_mem_rd_B <= 1'b0; 
cac_mem_wrt_B <= 1'b0; 
priority_bus_bus_req_B <= 1'b0; 
snoop_B <= 1'b0; 
snoop_page_1 <= 1'b0;
snoop_page_2 <= 1'b0; 
invalidate_B <= 1'b0;
read_B <= 1'b0; 
write_B <= 1'b0; 
dirty <= 1'b0;
priority_wrt_B <= 1'b0; 
bus_bus_bus_req_B <= 1'b0;

//If it is a read
if(cpu_cac_rd_B == 1'b1) 
begin
read_B <= 1'b1;
if(cpu_tag == cur_tag_1) 
begin
if(valid_1 == 1'b0)
begin
$display ("Read Miss In Invalid State PAGE 2"); 
cac_cpu_miss_B <= 1'b1;
bus_bus_bus_req_B <= 1'b1; 
state <= S1;
end
//read hit in EXCLUSIVE state

if((dirty_1 == 1'b1) & (valid_1 == 1'b1)) 
begin
$display ("Read Hit In Exculsive State PAGE 2"); 
cac_cpu_hit_B <= 1'b1;
cac_cpu_data_B <= cpu_buffer_1[4:0];
buffer_1[cpu_index] <= buffer_1[cpu_index] | 11'b00010000000; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111; 
state <= S0;
end
//read hit in SHARED state

if((dirty_1 == 1'b0) & (valid_1 == 1'b1)) 
begin
$display ("Read Hit In Shared State PAGE 2"); 
cac_cpu_hit_B <= 1'b1;
cac_cpu_data_B <= cpu_buffer_1[4:0];
buffer_1[cpu_index] <= buffer_1[cpu_index] | 11'b00010000000; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111;
state <= S0; 
end
end

if(cpu_tag == cur_tag_2) //Check Page 2 of the Cache 
begin
if(valid_2 == 1'b0) 
begin
$display ("Read Miss In Invalid State PAGE 2");
cac_cpu_miss_B <= 1'b1;
bus_bus_bus_req_B <= 1'b1; state <= S1;
end

//read hit in EXCLUSIVE state 
if((dirty_2 == 1'b1) & (valid_2 == 1'b1)) 
begin
$display ("Read Hit In Exculsive State PAGE 2"); 
cac_cpu_hit_B <= 1'b1;
cac_cpu_data_B <= cpu_buffer_2[4:0];
buffer_2[cpu_index] <= buffer_2[cpu_index] | 11'b00010000000; 
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111;
state <= S0; 
end

if((dirty_2 == 1'b0) & (valid_2 == 1'b1)) 
begin
$display ("Read Hit In Shared State PAGE 2"); 
cac_cpu_hit_B <= 1'b1;
cac_cpu_data_B <= cpu_buffer_2[4:0];
buffer_2[cpu_index] <= buffer_2[cpu_index] | 11'b00010000000; 
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111; 
state <= S0;
end
end

if((cpu_tag != cur_tag_1) & (cpu_tag != cur_tag_2)) 
begin
$display ("Read Miss In Invalid State"); 
cac_cpu_miss_B <= 1'b1;
bus_bus_bus_req_B <= 1'b1;
state <= S1; 
end
end

if(cpu_cac_wrt_B == 1'b1) 
begin
write_B <= 1'b1; 
if(cpu_tag == cur_tag_1) 
begin

//write miss in INVALID state 
if(valid_1 == 1'b0) 
begin
$display ("Write Miss In Invalid State PAGE 2"); 
cac_cpu_miss_B <= 1'b1;
bus_bus_bus_req_B <= 1'b1; 
state <= S1;
end

//write miss in EXCLUSIVE state
else if((dirty_1 == 1'b1) & (valid_1 == 1'b1)) 
begin
$display ("Write Hit In Exculsive State PAGE 2"); 
cac_cpu_hit_B <= 1'b1;
buffer_1[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_B}; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111; 
state <= S0;
end

//write hit in SHARED state
else if((dirty_1 == 1'b0) & (valid_1 == 1'b1)) 
begin
$display ("Write Hit In Shared State PAGE 2");
buffer_1[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_B}; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111; 
bus_bus_bus_req_B <= 1'b1;
back_t_s <= S0; 
state <= S4;
end
end

else if(cpu_tag == cur_tag_2) 
begin
if(valid_2 == 1'b0) 
begin
$display ("Write Miss In Invalid State PAGE 2"); 
cac_cpu_miss_B <= 1'b1;
bus_bus_bus_req_B <= 1'b1; 
state <= S1;
end

//write hit in EXCLUSIVE state
else if((dirty_2 == 1'b1) & (valid_2 == 1'b1)) 
begin
$display ("Write Hit In Exculsive State PAGE 2"); 
cac_cpu_hit_B <= 1'b1;
buffer_2[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_B};
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111;
state <= S0;
end

//write hit in SHARED state
else if((dirty_2 == 1'b0) & (valid_2 == 1'b1)) 
begin
$display ("Write Hit In Shared State PAGE 2");
buffer_2[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_B};
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111; 
bus_bus_bus_req_B <= 1'b1;
back_t_s <= S0; 
state <= S4;
end
end

else if((cpu_tag != cur_tag_1) & (cpu_tag != cur_tag_2)) 
begin
$display ("Write Miss In Invalid state"); 
cac_cpu_miss_B <= 1'b1; 
bus_bus_bus_req_B <= 1'b1;
state <= S1;
end
end

//if snooping
 
if(snoop_A == 1'b1) 
begin
if((snoop_tag == snoop_buffer_1[6:5])&(snoop_buffer_1[10] == 1'b1)) 
begin
if(snoop_buffer_1[9] == 1'b1) 
begin
priority_bus_bus_req_B <= 1'b1; 
snoop_page_1 <= 1'b1; 
back_t_s <= S0;
state <= S3; 
end

else 
begin
state <= S0; 
end
end

else if((snoop_tag == snoop_buffer_2[6:5])&(snoop_buffer_2[10] == 1'b1)) 
begin
if(snoop_buffer_2[9] == 1'b1) 
begin
priority_bus_bus_req_B <= 1'b1; 
snoop_page_2 <= 1'b1; 
back_t_s <= S0;
state <= S3; 
end

else 
begin
state <= S0; 
end
end
else 
begin
state <= S0; 
end
end

//if Invalidation

if(invalidate_A == 1'b1) 
begin
if((snoop_tag == snoop_buffer_1[6:5])&(snoop_buffer_1[10] == 1'b1)) 
begin
buffer_1[snoop_index] <= (buffer_1[snoop_index] & 11'b00101111111); 
buffer_2[snoop_index] <= (buffer_2[snoop_index] | 11'b00010000000);
state <= S0; 
end

else if((snoop_tag == snoop_buffer_2[6:5])&(snoop_buffer_1[10] == 1'b1)) 
begin
buffer_2[snoop_index] <= (buffer_2[snoop_index] & 11'b00101111111); 
buffer_1[snoop_index] <= (buffer_1[snoop_index] | 11'b00010000000);
state <= S0;
end
else 
begin
state <= S0; 
end
end
end //State 0

//State S0 - Wait

S1:

begin

cac_cpu_miss_B <= 1'b0; priority_wrt_B <= 1'b0; 
$display ("Waiting in S1"); 
if(bus_B == 1'b1)
begin
cac_mem_rd_B <= 1'b1; 
cac_memAdd_B <= add_mem; 
snoop_B <= 1'b1; 
snoop_add_B <= add_mem; 
bus_bus_bus_req_B <= 1'b0; 
state <= S2;
end

//if invalidation
else if(invalidate_A == 1'b1) 
begin
if((snoop_tag == snoop_buffer_1[6:5])&(snoop_buffer_1[10] == 1'b1)) 
begin
buffer_1[snoop_index] <= (buffer_1[snoop_index] & 11'b00101111111);
buffer_2[snoop_index] <= (buffer_2[snoop_index] | 11'b00010000000);
if(snoop_add_A == cpu_cac_add_B) 
begin
bus_bus_bus_req_B <= 1'b0; 
cac_cpu_miss_B <= 1'b1; 
state <= S1;
end 
else 
begin
state <= S1; 
end

end

else if((snoop_tag == snoop_buffer_2[6:5])&(snoop_buffer_2[10] == 1'b1)) begin

buffer_2[snoop_index] <= (buffer_2[snoop_index] & 11'b00101111111);
 buffer_1[snoop_index] <= (buffer_1[snoop_index] | 11'b00010000000);

if(snoop_add_A == cpu_cac_add_B) begin

bus_bus_bus_req_B <= 1'b0; cac_cpu_miss_B <= 1'b1; state <= S1;

end else begin

state <= S1; end
end

else
begin

state <= S1; end

end else begin

state <= S1; end
end //S1

//State S2
 
S2: begin

dirty <= 1'b0; 
snoop_B <= 1'b0;
cac_mem_wrt_B <= 1'b0; 
cac_mem_rd_B <= 1'b0; 
snoop_page_1 <= 1'b0; 
snoop_page_2 <= 1'b0; 
priority_wrt_B <= 1'b0;

if((data_avail_memB == 1'b1)|(dirty == 1'b1)) 
begin
if(update_1 == 1'b0) 
begin
if(dirty_1 == 1'b1) 
begin
cac_mem_wrt_B <= 1'b1; 
cac_memAdd_B <= add_mem;
cac_mem_data_B <= cpu_cac_data_B;
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b10111111111; 
dirty <= 1'b1;
state <= S2; 
end

else if(dirty_1 == 1'b0) 
begin
buffer_1[cpu_index] <= {1'b1,1'b0,1'b0,1'b1,cpu_tag,mem_cac_data_B}; 
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b11101111111; 
if((read_B == 1'b1) & (write_B == 1'b0))
begin
state <= S0; 
end

else if((read_B == 1'b0) & (write_B == 1'b1)) 
begin
buffer_1[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_B}; 
cac_mem_wrt_B <= 1'b1;
cac_memAdd_B <= add_mem; 
cac_mem_data_B <= cpu_cac_data_B; 
invalidate_B <= 1'b1;
snoop_add_B <= add_mem; 
state <= S0;
end
end 
end

else if(update_2 == 1'b0) 
begin
if(dirty_2 == 1'b1) 
begin
cac_mem_wrt_B <= 1'b1; 
cac_memAdd_B <= add_mem; 
cac_mem_data_B <= cpu_cac_data_B;
buffer_2[cpu_index] <= buffer_2[cpu_index] & 11'b10111111111; 
dirty <= 1'b1;
state <= S2;
end

else if(dirty_2 == 1'b0) 
begin
buffer_2[cpu_index] <= {1'b1,1'b0,1'b0,1'b1,cpu_tag,mem_cac_data_B}; 
buffer_1[cpu_index] <= buffer_1[cpu_index] & 11'b11101111111; 
if((read_B == 1'b1) & (write_B == 1'b0))
begin
state <= S0; 
end
 
if((read_B == 1'b0) & (write_B == 1'b1)) 
begin
buffer_2[cpu_index] <= {1'b1,1'b1,1'b0,1'b1,cpu_tag,cpu_cac_data_B}; 
cac_mem_wrt_B <= 1'b1;
cac_memAdd_B <= add_mem; 
cac_mem_data_B <= cpu_cac_data_B; 
invalidate_B <= 1'b1;
snoop_add_B <= add_mem; 
state <= S0;
end
end 
end
end

//if snooping
else if(snoop_A == 1'b1) 
begin
if((snoop_tag == snoop_buffer_1[6:5]) & (snoop_buffer_1[10] == 1'b1)) 
begin
if(snoop_buffer_1[9] == 1'b1) 
begin
priority_bus_bus_req_B <= 1'b1; 
snoop_page_1 <= 1'b1; 
back_t_s <= S2;
state <= S3; 
end
else 
begin
state <= S2; 
end
end

else if((snoop_tag == snoop_buffer_2[6:5]) & (snoop_buffer_2[10] == 1'b1)) 
begin
if(snoop_buffer_2[9] == 1'b1) 
begin
priority_bus_bus_req_B <= 1'b1; 
snoop_page_2 <= 1'b1; 
back_t_s <= S2;
state <= S3; 
end
else 
begin
state <= S2; 
end
end
else 
begin
state <= S2; 
end
end

//if invalidation
else if(invalidate_A == 1'b1) 
begin
if((snoop_tag == snoop_buffer_1[6:5])& (snoop_buffer_1[10] == 1'b1)) 
begin
buffer_1[snoop_index] <= buffer_1[snoop_index] & 11'b00110111111; 
buffer_2[snoop_index] <= buffer_2[snoop_index] | 11'b00001000000; 
state <= S2;
end

else if((snoop_tag == snoop_buffer_2[6:5]) & (snoop_buffer_2[10] == 1'b1)) 
begin
buffer_2[snoop_index] <= buffer_2[snoop_index] & 11'b00110111111; 
buffer_1[snoop_index] <= buffer_1[snoop_index] | 11'b00001000000;
state <= S2;
end
else 
begin
state <= S2; 
end
end 
end // S2

// State S3

S3: 
begin
if(bus_B == 1'b1)
begin
priority_bus_bus_req_B <= 1'b0;
if((snoop_page_1 == 1'b1)&(snoop_page_2 == 1'b0))
begin
priority_wrt_B <= 1'b1; cac_memAdd_B <= snoop_add_A;
cac_mem_data_B <= snoop_buffer_1[4:0];
buffer_1[snoop_index] <= buffer_1[snoop_index] & 11'b10111111111; 
state <= S5;
end
else if((snoop_page_1 == 1'b0)&(snoop_page_2 == 1'b1)) 
begin
priority_wrt_B <= 1'b1; 
cac_memAdd_B <= snoop_add_A;
cac_mem_data_B <= snoop_buffer_2[4:0];
buffer_2[snoop_index] <= buffer_2[snoop_index] & 11'b10111111111; 
state <= S5;
end
end
else
begin
state <= S3; 
end
end//S3

//State S4

S4: 
begin
if(bus_B == 1'b1) 
begin
bus_bus_bus_req_B <= 1'b0; 
cac_cpu_hit_B <= 1'b1;
invalidate_B <= 1'b1; 
snoop_add_B <= add_mem;
cac_mem_wrt_B <= 1'b1;
cac_memAdd_B <= add_mem; 
cac_mem_data_B <= cpu_cac_data_B; 
state <= back_t_s;
end
end // S4

//State S5
S5: 
begin
state <= back_t_s; 
end
endcase 
end
end
endmodule


// Main Memory

module Memory (
clock, rst_1,
cac_data_avail_mem_Add_A, 
cac_mem_data_A, 
cac_mem_rd_A, 
cac_mem_wrt_A, 
cac_data_avail_mem_Add_B, 
cac_mem_data_B, 
cac_mem_rd_B, 
cac_mem_wrt_B, 
mem_cac_data_B, 
mem_cac_data_A, 
bus_bus_mem, 
priority_wrt_A, 
priority_wrt_B,
memA,
memB, 
bus_mem);

//Inputs
input clock, rst_1;
input cac_mem_rd_A,cac_mem_wrt_A,cac_mem_rd_B,cac_mem_wrt_B; 
input [4:0] cac_data_avail_mem_Add_A;
input [4:0] cac_data_avail_mem_Add_B; 
input [4:0] cac_mem_data_A;
input [4:0] cac_mem_data_B; 
input priority_wrt_A;
input priority_wrt_B; 
input bus_mem;

//Outputs
output [4:0] mem_cac_data_A; 
output [4:0] mem_cac_data_B; 
output bus_bus_mem;
output memA; 
output memB;

//Registers
reg [0:9] memArray [31:0]; 
reg [4:0] mem_cac_data_A; 
reg [4:0] mem_cac_data_B; 
reg [4:0] mem_cache_data; 
reg memA;
reg memB;
reg bus_bus_mem; 
reg ready_bit_A; 
reg ready_bit_B; 
reg nextA;
reg nextB; 
reg [2:0] state; 
reg [4:0] add;
reg [4:0] next_add;

//Internals
parameter S0 = 0; 
parameter S1 = 1; 
parameter S2 = 2; 
parameter S3 = 3; 
parameter S4 = 4;

//Memory
always @(posedge clock or negedge rst_1) 
begin
if (~rst_1) 
begin
memArray[31] = 10'b1111100000; 
memArray[30] = 10'b1111000001; 
memArray[29] = 10'b1110100010; 
memArray[28] = 10'b1110000011; 
memArray[27] = 10'b1101100100; 
memArray[26] = 10'b1101000101; 
memArray[25] = 10'b1100100110; 
memArray[24] = 10'b1100000111; 
memArray[23] = 10'b1011101000; 
memArray[22] = 10'b1011001001; 
memArray[21] = 10'b1010101010; 
memArray[20] = 10'b1010001011; 
memArray[19] = 10'b1001101100; 
memArray[18] = 10'b1001001101; 
memArray[17] = 10'b1000101110; 
memArray[16] = 10'b1000001111; 
memArray[15] = 10'b0111110000; 
memArray[14] = 10'b0111010001; 
memArray[13] = 10'b0110110010; 
memArray[12] = 10'b0110010011; 
memArray[11] = 10'b0101110100; 
memArray[10] = 10'b0101010101; 
memArray[9] = 10'b0100110110; 
memArray[8] = 10'b0100010111; 
memArray[7] = 10'b0011111000; 
memArray[6] = 10'b0011011001; 
memArray[5] = 10'b0010111010; 
memArray[4] = 10'b0010011011; 
memArray[3] = 10'b0001111100; 
memArray[2] = 10'b0001011101; 
memArray[1] = 10'b0000111110; 
memArray[0] = 10'b0000011111;
mem_cac_data_A <= 5'b0; 
mem_cac_data_B <= 5'b0; 
memA <= 1'b0;
memB <= 1'b0; 
nextA <= 1'b0; 
nextB <= 1'b0; 
add <= 5'b0; 
state <= S0;
bus_bus_mem <= 1'b0; 
end
else
begin
state <= S0; 
end

case(state)
S0: 
begin
memA <= 1'b0; 
memB <= 1'b0; 
ready_bit_A <= 1'b0; 
ready_bit_B <= 1'b0; 
bus_bus_mem <= 1'b0; 
add <= 5'b0;

// IF WRITE
if (priority_wrt_A == 1'b1) 
begin
memArray[cac_data_avail_mem_Add_A] <= {cac_data_avail_mem_Add_A, cac_mem_data_A}; 
state <= S0;
end
else if (priority_wrt_B == 1'b1) 
begin
memArray[cac_data_avail_mem_Add_B] <= {cac_data_avail_mem_Add_B, cac_mem_data_B}; 
state <= S0;
end
else if ((cac_mem_wrt_A == 1'b1) & (cac_mem_rd_A == 1'b0)) 
begin
memArray[cac_data_avail_mem_Add_A] <= {cac_data_avail_mem_Add_A, cac_mem_data_A}; 
state <= S0;
end
else if ((cac_mem_wrt_B == 1'b1) & (cac_mem_rd_B == 1'b0)) 
begin
memArray[cac_data_avail_mem_Add_B] <= {cac_data_avail_mem_Add_B, cac_mem_data_B}; 
state <= S0;
end

// FINISH WRITE
else if ((cac_mem_wrt_A == 1'b0) & ((cac_mem_rd_A == 1'b1)|(nextA == 1'b1))) 
begin
if ((cac_mem_rd_A == 1'b1)&(nextA == 1'b0)) 
begin
add <= cac_data_avail_mem_Add_A; ready_bit_A <= cac_mem_rd_A; 
state <= S1;
end
else if ((cac_mem_rd_A == 1'b0)&(nextA == 1'b1)) 
begin
add <= next_add; 
ready_bit_A <= 1'b1; 
nextA <= 1'b0;
state <= S1; 
end
end
else if ((cac_mem_wrt_B == 1'b0) & ((cac_mem_rd_B == 1'b1)|(nextB == 1'b1)))
begin
if ((cac_mem_rd_B == 1'b1)&(nextB == 1'b0)) 
begin
add <= cac_data_avail_mem_Add_B; 
ready_bit_B <= cac_mem_rd_B; 
state <= S1;
end
else if ((cac_mem_rd_B == 1'b0)&(nextB == 1'b1)) 
begin
add <= next_add; 
ready_bit_B <= 1'b1; 
nextB <= 1'b0;
state <= S1; 
end
end
else 
begin
state <= S0; 
end
end//S0

S1: 
begin
bus_bus_mem <= 1'b1; 
if(priority_wrt_A == 1'b1)
begin
memArray[cac_data_avail_mem_Add_A] <= {cac_data_avail_mem_Add_A, cac_mem_data_A}; 
state <= S2;

end

else if (priority_wrt_B == 1'b1) begin

memArray[cac_data_avail_mem_Add_B] <= {cac_data_avail_mem_Add_B, cac_mem_data_B}; state <= S2;
end

else if(cac_mem_wrt_A == 1'b1) begin

memArray[cac_data_avail_mem_Add_A] <= {cac_data_avail_mem_Add_A, cac_mem_data_A}; state <= S2;

end

else if (cac_mem_wrt_B == 1'b1) begin

memArray[cac_data_avail_mem_Add_B] <= {cac_data_avail_mem_Add_B, cac_mem_data_B}; state <= S2;
end

else begin

state <= S2; end

end//S0

S2: begin

if(priority_wrt_A == 1'b1)
 
 
begin

memArray[cac_data_avail_mem_Add_A] <= {cac_data_avail_mem_Add_A, cac_mem_data_A}; state <= S2;

end

else if (priority_wrt_B == 1'b1) begin

memArray[cac_data_avail_mem_Add_B] <= {cac_data_avail_mem_Add_B, cac_mem_data_A}; state <= S2;
end

else if (cac_mem_rd_A == 1'b1) begin
nextA <= 1'b1;

next_add <= cac_data_avail_mem_Add_A; state <= S2;

end

else if (cac_mem_rd_B == 1'b1) begin

nextB <= 1'b1;

next_add <= cac_data_avail_mem_Add_B; state <= S2;

end

else if (cac_mem_wrt_A == 1'b1) begin

memArray[cac_data_avail_mem_Add_A] <= {cac_data_avail_mem_Add_A, cac_mem_data_A}; state <= S2;
end

else if (cac_mem_wrt_B == 1'b1) 
begin
memArray[cac_data_avail_mem_Add_B] <= {cac_data_avail_mem_Add_B, cac_mem_data_B}; 
state <= S2;
end
else if (bus_mem == 1'b1) 
begin
bus_bus_mem <= 1'b0;
if ((ready_bit_A == 1'b1)&(ready_bit_B == 1'b0)) 
begin
mem_cac_data_A <= memArray[add]; 
memA <= 1'b1;
state <= S0; 
end
if ((ready_bit_A == 1'b0)&(ready_bit_B == 1'b1)) 
begin
mem_cac_data_B <= memArray[add]; 
memB <= 1'b1;
state <= S0; 
end
end
else 
begin
state <= S2; 
end
end//S2
endcase
end 
endmodule

// Memory Mapping Unit

module MMU ( clock,rst_1, 
cpu_cac_add_A, 
cpu_cac_add_B, 
cpu_cac_rd_A, 
cpu_cac_rd_B, 
cpu_cac_wrt_A, 
cpu_cac_wrt_B, 
cpu_cac_data_A, 
cpu_cac_data_B, 
cpu_mmu_add_A, 
cpu_mmu_add_B, 
cpu_mmu_rd_A, 
cpu_mmu_rd_B, 
cpu_mmu_wrt_A, 
cpu_mmu_wrt_B,
cpu_mmu_data_A, 
cpu_mmu_data_B
);
//Inputs

input clock, rst_1;

input cpu_mmu_rd_A, cpu_mmu_rd_B; input wire cpu_mmu_wrt_A, cpu_mmu_wrt_B; input [4:0] cpu_mmu_data_A;

input [4:0] cpu_mmu_data_B; input [6:0] cpu_mmu_add_A; input [6:0] cpu_mmu_add_B;

//Outputs

output cpu_cac_rd_A, cpu_cac_rd_B; output cpu_cac_wrt_A, cpu_cac_wrt_B; output [4:0] cpu_cac_data_A;

output [4:0] cpu_cac_data_B; output [4:0] cpu_cac_add_A; output [4:0] cpu_cac_add_B;

//Registers

reg cpu_cac_rd_A, cpu_cac_rd_B;  reg [4:0] cpu_cac_data_A; reg cpu_cac_wrt_A,cpu_cac_wrt_B;

reg [4:0] cpu_cac_data_B; reg [4:0] cpu_cac_add_A; reg [4:0] cpu_cac_add_B;

//Internals


wire clock, rst_1, cpu_mmu_rd_A, cpu_mmu_rd_B;

//Begin//

always @(posedge clock or negedge rst_1) begin

if (~rst_1) begin //initials

cpu_cac_add_A <= 5'b0; 
cpu_cac_add_B <= 5'b0; 
cpu_cac_data_A <= 5'b0; 
cpu_cac_data_B <= 5'b0; 
cpu_cac_rd_A <= 1'b0; 
cpu_cac_rd_B <= 1'b0; 
cpu_cac_wrt_A <= 1'b0; 
cpu_cac_wrt_B <= 1'b0;
end

else begin

cpu_cac_wrt_A <= cpu_mmu_wrt_A; 
cpu_cac_wrt_B <= cpu_mmu_wrt_B;
cpu_cac_rd_A <= cpu_mmu_rd_A; 
cpu_cac_rd_B <= cpu_mmu_rd_B; 
cpu_cac_data_A <= cpu_mmu_data_A;
cpu_cac_data_B <= cpu_mmu_data_B; 
cpu_cac_add_A <= cpu_mmu_add_A[4:0]; 
cpu_cac_add_B <= cpu_mmu_add_B[4:0];

end end

endmodule

// Memory Bus Controller

module MemoryBusController(

clock, rst_1,

bus_bus_req_A, bus_bus_req_B, bus_req_mem, bus_mem, bus_A,

bus_B, priority_bus_req_A, priority_bus_req_B );

// Inputs

input clock; 
input rst_1;
input bus_bus_req_A; 
input bus_bus_req_B; 
input priority_bus_req_A; 
input priority_bus_req_B; 
input bus_req_mem;

//Outputs

output reg bus_A;
output reg bus_B; 
output reg bus_mem;

//Registers

reg[2:0] Bus_state;

//Internals

wire clock; wire rst_1;

wire bus_bus_req_A; wire bus_bus_req_B; wire priority_bus_req_A; wire priority_bus_req_B; 

//Parameters

parameter S0 = 0; //Initial

parameter S1 = 1; //Granting the bus to the Cache A 
parameter S2 = 2; //Granting the bus to the Cache B
parameter S3 = 3; //Granting the bus to the memory 
parameter S4 = 4; //Wait
 

// Module

always @(negedge clock or negedge rst_1) begin

if(rst_1 == 0) begin //Initials

Bus_state <= S0; bus_A <= 1'b0; bus_B <= 1'b0; bus_mem <= 1'b0; end

else

case(Bus_state)

S0: //Initial 
begin

bus_A <= 1'b0; bus_B <= 1'b0; bus_mem <= 1'b0;

if (priority_bus_req_A == 1'b1) //Priority Cache A 
begin

Bus_state <= S1; end

else if (priority_bus_req_B == 1'b1) //Priority Cache B
 begin

Bus_state <= S2; end

else if (bus_bus_req_A == 1'b0 & bus_bus_req_B ==1'b0 & bus_req_mem ==1'b1) //Memory 
begin

Bus_state <= S3; end

else if (bus_bus_req_A ==1'b1 & bus_bus_req_B == 1'b0 & bus_req_mem == 1'b1) begin

Bus_state <= S3; end

else if (bus_bus_req_A == 1'b0 & bus_bus_req_B == 1'b1 & bus_req_mem == 1'b1) begin

Bus_state <= S3; end

else if (bus_bus_req_A == 1'b1 & bus_bus_req_B == 1'b1 & bus_req_mem == 1'b1) begin

Bus_state <= S3; end

else if (bus_bus_req_A == 1'b1 & bus_bus_req_B == 1'b0 & bus_req_mem == 1'b0) begin

Bus_state <= S1; end

else if (bus_bus_req_A == 1'b0 & bus_bus_req_B == 1'b1 & bus_req_mem == 1'b0) begin

Bus_state <= S2; end

else if (bus_bus_req_A == 1'b1 & bus_bus_req_B == 1'b1 & bus_req_mem == 1'b0) begin

Bus_state <= S1; end

else begin

Bus_state <= S0; end
end

S1: //For Cache A 
begin

bus_A <= 1'b1; Bus_state <= S0;

end

S2: //For Cache B
 begin

bus_B <= 1'b1; Bus_state <= S0;
end

S3: //For memory 
begin

bus_mem <= 1'b1; Bus_state <= S0;

end

S4: begin

bus_A <= 1'b0; bus_B <= 1'b0; bus_mem <= 1'b0; Bus_state <= S0;

end endcase

end 
endmodule

// Test Bench

// Verilog Test Fixture Template

module Test();

//Inputs
reg clock, rst_l, cpu_mmu_rd_A, cpu_mmu_wrt_A; 
reg cpu_mmu_rd_B, cpu_mmu_wrt_B;
reg [4:0] cpu_mmu_data_A; 
reg [4:0] cpu_mmu_data_B; 
reg [6:0] cpu_mmu_add_A; 
reg [6:0] cpu_mmu_add_B;

// Cache - Cpu

wire cac_cpu_hit_A; 
wire cac_cpu_miss_A; 
wire [4:0] cac_cpu_data_A; 
wire cac_cpu_hit_B;
wire cac_cpu_miss_B; 
wire [4:0] cac_cpu_data_B;

// Cache – Memory Bus Controller

wire bus_req_A, priority_req_A; 
wire bus_req_B, priority_req_B; 
wire bus_B, bus_mem, bus_A;
wire bus_req_mem;
// Cache - Memory

wire cac_mem_rd_A, cac_mem_wrt_A, data_avail_mem_A, priority_wrt_A; 
wire cac_mem_rd_B, cac_mem_wrt_B, priority_wrt_B, data_avail_mem_B; 
wire [4:0] cac_mem_add_A, cac_mem_data_A, mem_cac_data_A;
wire [4:0] cac_mem_add_B, cac_mem_data_B, mem_cac_data_B;

// MMU - Cache

wire cpu_cac_rd_A, cpu_cac_wrt_A; 
wire cpu_cac_rd_B, cpu_cac_wrt_B;
wire [4:0] cpu_cac_add_A, cpu_cac_data_A; 
wire [4:0] cpu_cac_add_B, cpu_cac_data_B;

//Cache A - B

wire snoop_A, snoop_B;
wire invalidate_A, invalidate_B; 
wire [4:0] snoop_add_A;
wire [4:0] snoop_add_B;

// Instantiate Memory

Memory memo(clock, rst_l, cac_mem_add_A, cac_mem_data_A, cac_mem_rd_A, cac_mem_wrt_A, cac_mem_add_B, cac_mem_data_B, 
cac_mem_rd_B, cac_mem_wrt_B, mem_cac_data_B, mem_cac_data_A, bus_req_mem, priority_wrt_A, priority_wrt_B, 
data_avail_mem_A, data_avail_mem_B,bus_mem);
 

//Instantiate MMU

MMU mmu(clock, rst_l,cpu_cac_add_A, cpu_cac_add_B, cpu_cac_rd_A, cpu_cac_rd_B, cpu_cac_wrt_A, cpu_cac_wrt_B, 
cpu_cac_data_A, cpu_cac_data_B, cpu_mmu_add_A, cpu_mmu_add_B, cpu_mmu_rd_A, cpu_mmu_rd_B, cpu_mmu_wrt_A, cpu_mmu_wrt_B, 
cpu_mmu_data_A, cpu_mmu_data_B);


//Instantiate CacheA

Cache_A cache(clock, rst_l, cpu_cac_add_A, cpu_cac_rd_A, cpu_cac_wrt_A, cpu_cac_data_A, mem_cac_data_A, 
data_avail_mem_A, bus_req_A, priority_req_A, snoop_B, snoop_add_B, invalidate_B,cac_cpu_hit_A,cac_cpu_miss_A, cac_cpu_data_A, cac_mem_add_A, cac_mem_data_A, cac_mem_rd_A, cac_mem_wrt_A, priority_wrt_A,bus_A, snoop_A, snoop_add_A, invalidate_A);

//Instantiate CacheB

Cache_B cacheb(clock, rst_l, cpu_cac_add_B, cpu_cac_rd_B, cpu_cac_wrt_B, cpu_cac_data_B, mem_cac_data_B, data_avail_mem_B, bus_req_B, priority_req_B, snoop_A, snoop_add_A, invalidate_A,cac_cpu_hit_B,cac_cpu_miss_B, cac_cpu_data_B, cac_mem_add_B, cac_mem_data_B, cac_mem_rd_B, cac_mem_wrt_B, priority_wrt_B,bus_B, snoop_B, snoop_add_B, invalidate_B);

//Instantiate MemoryBusController

MemoryBusController mbc(clock, rst_l, bus_req_A, bus_req_B, bus_req_mem, bus_mem, bus_A, bus_B, priority_req_A, priority_req_B);

always 
begin
#5 clock <= ~clock;
end

// Start
initial
begin
clock	<= 1'b0;
rst_l	<= 1'b1;
cpu_mmu_rd_A <= 1'b0;
cpu_mmu_rd_B <= 1'b0;
cpu_mmu_wrt_A <= 1'b0;
cpu_mmu_wrt_B <= 1'b0;
cpu_mmu_add_A <= 7'b0; 
cpu_mmu_add_B <= 7'b0; 
cpu_mmu_data_A <= 5'b0; 
cpu_mmu_data_B <= 5'b0;

# 10
rst_l	<= 1'b0;

# 10
rst_l	<= 1'b1;

# 10

cpu_mmu_rd_A <= 1'b1; 
cpu_mmu_add_A <= 7'b0011101; 
cpu_mmu_rd_B <= 1'b1; 
cpu_mmu_add_B <= 7'b1110011;

# 10

cpu_mmu_rd_A <= 1'b0; 
cpu_mmu_rd_B <= 1'b0;

#	100 

#	10 

cpu_mmu_rd_A <= 1'b1; cpu_mmu_add_A <= 7'b1000101;
cpu_mmu_rd_B <= 1'b1; cpu_mmu_add_B <= 7'b0101010;

# 10

cpu_mmu_rd_A <= 1'b0; 
cpu_mmu_rd_B <= 1'b0;

# 100

#10

cpu_mmu_rd_A <= 1'b1; 
cpu_mmu_add_A <= 7'b0011101; 
cpu_mmu_rd_B <= 1'b1; 
cpu_mmu_add_B <= 7'b1110011;

# 10

cpu_mmu_rd_A <= 1'b0; 
cpu_mmu_rd_B <= 1'b0;

end
endmodule


