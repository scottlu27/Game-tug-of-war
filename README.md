# Game-tug-of-war
module tug_of_war (
input             sys_clk   ,
input             sys_rst_n ,
input             left_btn,
input             right_btn,                        
output  wire      Hs        ,
output  wire      Vs        ,
output sync, blank,clk_vga,  
output  [7:0]     VGA_G     ,
output  [7:0]     VGA_B     ,
output  [7:0]     VGA_R      );


//Wire define 
wire       valid_mov;
wire       move_right,move_left ;
reg clk_25m;

always@(posedge sys_clk)
    clk_25m <= ~clk_25m;

main_control main_control(
clk_25m   ,
sys_rst_n ,
move_left,
move_right,
clk_vga   ,        
Hs        ,
Vs        ,
sync,
blank,    
VGA_G     ,
VGA_B     ,
VGA_R      );

button_module button_module(
clk_vga,
sys_rst_n,
left_btn,
right_btn,
move_left,
move_right
);
endmodule  
module button_module(
input clk,
input rst_n,
input left_btn,
input right_btn,
output reg move_left,
output reg move_right
);

reg [31:0] cycle_cnt; // count for 1s
reg left_btn_r, right_btn_r;  //button registers
reg [7:0] left_cnt, right_cnt; //how many times bottons have been pressed during 1 second

always@(posedge clk)
begin
    left_btn_r <= left_btn;
    right_btn_r <= right_btn;
end

always@(posedge clk)
begin
    if(~rst_n)
        left_cnt <= 0;
    else
    begin
        if(cycle_cnt == 32'd25000000)
            left_cnt <= 0;
        else if({left_btn_r, left_btn} == 2'b01)
            left_cnt <= left_cnt + 1;
    end
end

always@(posedge clk)
begin
    if(~rst_n)
        right_cnt <= 0;
    else
    begin
        if(cycle_cnt == 32'd50000000)
            right_cnt <= 0;
        else if({right_btn_r, right_btn} == 2'b01)
            right_cnt <= right_cnt + 1;
    end
end


//count 1s, update state per 1s
always@(posedge clk)
begin
    if(~rst_n)
        cycle_cnt <= 0;
    else
    begin
        if(cycle_cnt == 32'd50000000)
            cycle_cnt <= 0;
        else
            cycle_cnt <= cycle_cnt + 1;
    end
end

//output control signal
always@(posedge clk)
begin
    if(~rst_n)
    begin
        move_left <= 0;
        move_right <= 0;
    end
    else
    begin
        if(cycle_cnt == 32'd50000000)
        begin
            move_left <= (left_cnt > right_cnt) ? 1'b1 : 1'b0;
            move_right <= (left_cnt < right_cnt) ? 1'b1 : 1'b0;
        end
        else
        begin
            move_left <= 1'b0;
            move_right <= 1'b0;
        end
    end
end
endmodule  
module main_control (
input             iCLK   ,
input             iRST_N ,
input             move_left,
input             move_right,
output         clk_vga   ,        
output         Hs        ,
output         Vs        ,
output sync, blank, 
output  [7:0]     VGA_G     ,
output  [7:0]     VGA_B     ,
output  [7:0]     VGA_R      );

//Reg define

wire  [9:0]  x_pixel ;
wire  [9:0]  y_pixel ;

reg  [9:0] flag_x_pos;  
reg  [9:0] flag_y_pos;  

//Wire define
wire       flag_vld;
wire       left_vld  ;
wire       right_vld  ;
wire       middle_vld  ;

//	VGA Side
reg	oVGA_HS;
reg	oVGA_VS;

assign Hs = oVGA_HS;
assign Vs = oVGA_VS;

//	Internal Registers
reg	[10:0]	H_Cont;
reg	[10:0]	V_Cont;
////////////////////////////////////////////////////////////
//	Horizontal	Parameter
parameter	H_FRONT	=	16;
parameter	H_SYNC	=	96;
parameter	H_BACK	=	48;
parameter	H_ACT	=	640;
parameter	H_BLANK	=	H_FRONT+H_SYNC+H_BACK;
parameter	H_TOTAL	=	H_FRONT+H_SYNC+H_BACK+H_ACT;
////////////////////////////////////////////////////////////
//	Vertical Parameter
parameter	V_FRONT	=	11;
parameter	V_SYNC	=	2;
parameter	V_BACK	=	31;
parameter	V_ACT	=	480;
parameter	V_BLANK	=	V_FRONT+V_SYNC+V_BACK;
parameter	V_TOTAL	=	V_FRONT+V_SYNC+V_BACK+V_ACT;
////////////////////////////////////////////////////////////
assign	sync	=	1'b1;	//	This pin is unused.
assign	blank	=	~((H_Cont<H_BLANK)||(V_Cont<V_BLANK));
assign	clk_vga	=	~iCLK;
assign	x_pixel	=	(H_Cont>=H_BLANK)	?	H_Cont-H_BLANK	:	11'h0	;
assign	y_pixel	=	(V_Cont>=V_BLANK)	?	V_Cont-V_BLANK	:	11'h0	;

//	Horizontal Generator: Refer to the pixel clock
always@(posedge iCLK or negedge iRST_N)
begin
if(!iRST_N)
begin
H_Cont	<=	0;
oVGA_HS	<=	1;
end
else
begin
if(H_Cont<H_TOTAL)
H_Cont	<=	H_Cont+1'b1;
else
H_Cont	<=	0;
//	Horizontal Sync
if(H_Cont==H_FRONT-1)	//	Front porch end
oVGA_HS	<=	1'b0;
if(H_Cont==H_FRONT+H_SYNC-1)	//	Sync pulse end
oVGA_HS	<=	1'b1;
end
end
//|--------|
//____|-------------
//	Vertical Generator: Refer to the horizontal sync
always@(posedge oVGA_HS or negedge iRST_N)
begin
if(!iRST_N)
begin
V_Cont	<=	0;
oVGA_VS	<=	1;
end
else
begin
if(V_Cont<V_TOTAL)
V_Cont	<=	V_Cont+1'b1;
else
V_Cont	<=	0;
//	Vertical Sync
if(V_Cont==V_FRONT-1)	//	Front porch end
oVGA_VS	<=	1'b0;
if(V_Cont==V_FRONT+V_SYNC-1)	//	Sync pulse end
oVGA_VS	<=	1'b1;
end
end


    //flag
    always @ (posedge clk_vga) begin 
        if(~iRST_N)
        begin
            flag_x_pos <= 10'd320;
            flag_y_pos <= 10'd240;
        end
        else
        begin
            if(flag_x_pos == 10'd20 || flag_x_pos == 10'd620)
                flag_x_pos <= flag_x_pos;
            else if(move_left)
                flag_x_pos <= flag_x_pos - 30;
            else if(move_right)
                flag_x_pos <= flag_x_pos + 30;
        end
    end

    assign middle_vld = (x_pixel >= 10'd317&&x_pixel <= 321)?1'b1:1'b0;//middle_vld

    assign left_vld = (x_pixel >= 10'd17&&x_pixel <= 21)?1'b1:1'b0;//left_vld

    assign right_vld = (x_pixel >= 10'd617&&x_pixel <= 621)?1'b1:1'b0;//right_vld

    assign flag_vld = ((x_pixel < flag_x_pos + 10) &&  (x_pixel > flag_x_pos - 10) && (y_pixel >= 10'd230&&y_pixel <= 250)) ?1'b1:1'b0;  //flag_vld


                                      
    //--------------------assign VGA data-------------------------------------------//

    assign VGA_G =  ( middle_vld ) ? 8'h0F:8'b0;
    assign VGA_B =  ( flag_vld ) ? 8'h0F:8'b0;
    assign VGA_R =  ( left_vld | right_vld ) ? 8'h0F:8'b0;

endmodule  

