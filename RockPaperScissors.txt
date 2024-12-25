// Debouncer Module
module Debouncer(
    input wire clk,          
    input wire rst,        
    input wire btn_in,    
    output reg btn_out   
);
    // Debounce parameters
    parameter DB_MAX = 500000; // Approximately 10ms for a 50MHz clock
    
    reg [19:0] cnt;           // Counter
    reg sync0, sync1;         // Synchronization registers
    
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            cnt <= 0;
            sync0 <= 0;
            sync1 <= 0;
            btn_out <= 0;
        end else begin
            sync0 <= btn_in;
            sync1 <= sync0;
            
            if (sync1 == btn_out) begin
                cnt <= 0;
            end else begin
                cnt <= cnt + 1;
                if (cnt == DB_MAX) begin
                    btn_out <= sync1;
                    cnt <= 0;
                end
            end
        end
    end
endmodule

module SevenSegmentDecoder(
    input wire [3:0] digit,
    input wire game_over,
    output reg [6:0] segments,  // {g,f,e,d,c,b,a}
    output reg [1:0] COM        // 2-bit COM control, active LOW
);
    always @(*) begin
        if (game_over) begin
            case(digit)
                4'd0: begin    // Player wins
                    segments = 7'b0001100;  // 'P'
                    COM = 2'b10;           // Enable first display only
                end
                4'd1: begin    // Computer wins
                    segments = 7'b1000110;  // 'C'
                    COM = 2'b01;           // Enable second display only
                end
                default: begin
                    segments = 7'b1111111;  // All segments off
                    COM = 2'b11;           // All displays off
                end
            endcase
        end else begin
            // During normal gameplay, show score on first display
            COM = 2'b10;  // Enable first display only
            case(digit)
                4'd0: segments = 7'b1000000;  // 0
                4'd1: segments = 7'b1111001;  // 1
                4'd2: segments = 7'b0100100;  // 2
                4'd3: segments = 7'b0110000;  // 3
                4'd4: segments = 7'b0011001;  // 4
                4'd5: segments = 7'b0010010;  // 5
                default: segments = 7'b1111111; // All segments off
            endcase
        end
    end
endmodule

// Main RockPaperScissors Module
module RockPaperScissors(
    input wire clk,          
    input wire rst,       
    input wire btn_r,    //input for rock
    input wire btn_p,   //paper
    input wire btn_s, //scissors
    output reg led_r,     // show Player chose Rock
    output reg led_p,     // show Player chose Paper
    output reg led_s,     // show Player chose Scissors
    output reg led_cr,    // Computer chose Rock
    output reg led_cp,    // Computer chose Paper
    output reg led_cs,    // Computer chose Scissors
    output reg led_win,   // Player wins
    output reg led_lose,  // Player loses
    output reg led_tie,   // Tie
    output reg buzzer,    // Buzzer output for loss indication
    output wire [6:0] score_display,  // Seven segment display output
	  output wire [1:0] COM 
);
    wire db_r, db_p, db_s;
    
    // Score tracking
    reg [3:0] player_score;
    reg [3:0] comp_score;
    reg game_over;
    reg [3:0] display_value;
    
    // Instantiate Seven Segment Decoder
    SevenSegmentDecoder score_decoder(
        .digit(display_value),
        .game_over(game_over),
        .segments(score_display),
		  .COM(COM)
    );
    
    // Instantiate Debouncers
    Debouncer dbRock (
        .clk(clk),
        .rst(rst),
        .btn_in(btn_r),
        .btn_out(db_r)
    );
    
    Debouncer dbPaper (
        .clk(clk),
        .rst(rst),
        .btn_in(btn_p),
        .btn_out(db_p)
    );
    
    Debouncer dbScissors (
        .clk(clk),
        .rst(rst),
        .btn_in(btn_s),
        .btn_out(db_s)
    );
    
    // State Machine Definitions
    parameter IDLE = 2'd0;
    parameter PLAYER = 2'd1;
    parameter COMPUTER = 2'd2;
    parameter RESULT = 2'd3;
    
    reg [1:0] state, next;
    parameter ROCK = 2'd0;      
    parameter PAPER = 2'd1;     
    parameter SCISSORS = 2'd2;  
    parameter WIN = 2'd0;   
    parameter LOSE = 2'd1;  
    parameter TIE = 2'd2;    
    
    reg [1:0] p_choice;
    reg [1:0] c_choice;
    reg [1:0] result;
    reg [23:0] rand_cnt;
    reg [23:0] buzz_cnt;
    parameter BZ_DUR = 5000000;
    
    // Clock Process Logic
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            state <= IDLE;
            rand_cnt <= 0;
            buzz_cnt <= 0;
            game_over <= 0;
            player_score <= 0;
            comp_score <= 0;
            display_value <= 0;
            // Initialize all outputs
            led_r <= 0;
            led_p <= 0;
            led_s <= 0;
            led_cr <= 0;
            led_cp <= 0;
            led_cs <= 0;
            led_win <= 0;
            led_lose <= 0;
            led_tie <= 0;
            buzzer <= 0;
            p_choice <= ROCK;
            c_choice <= ROCK;
            result <= TIE;
        end else begin
            state <= next;
            rand_cnt <= rand_cnt + 1;
            
            // Update display value
            if (game_over) begin
                display_value <= (player_score > comp_score) ? 4'd0 : 4'd1; // 0 for P, 1 for C
            end else begin
                display_value <= player_score;
            end

            // Buzzer counter logic
            if (buzz_cnt >= BZ_DUR)
                buzz_cnt <= 0;
            else if (result == LOSE && buzz_cnt != BZ_DUR)
                buzz_cnt <= buzz_cnt + 1;
                
            // Game state logic
            case (state)
                IDLE: begin
                    if (!game_over) begin
                        // Clear all LEDs but maintain scores
                        led_r <= 0;
                        led_p <= 0;
                        led_s <= 0;
                        led_cr <= 0;
                        led_cp <= 0;
                        led_cs <= 0;
                        led_win <= 0;
                        led_lose <= 0;
                        led_tie <= 0;
                        buzzer <= 0;
                    end
                end
                PLAYER: begin
                    if (db_r) begin
                        p_choice <= ROCK;
                        led_r <= 1;
                        led_p <= 0;
                        led_s <= 0;
                    end else if (db_p) begin
                        p_choice <= PAPER;
                        led_r <= 0;
                        led_p <= 1;
                        led_s <= 0;
                    end else if (db_s) begin
                        p_choice <= SCISSORS;
                        led_r <= 0;
                        led_p <= 0;
                        led_s <= 1;
                    end
                end
                COMPUTER: begin
                    c_choice <= rand_cnt[2:1];
                    case (rand_cnt[2:1])
                        ROCK: begin
                            led_cr <= 1;
                            led_cp <= 0;
                            led_cs <= 0;
                        end
                        PAPER: begin
                            led_cr <= 0;
                            led_cp <= 1;
                            led_cs <= 0;
                        end
                        SCISSORS: begin
                            led_cr <= 0;
                            led_cp <= 0;
                            led_cs <= 1;
                        end
                        default: begin
                            led_cr <= 0;
                            led_cp <= 0;
                            led_cs <= 0;
                        end
                    endcase
                end
                RESULT: begin
                    if (p_choice == c_choice) begin
                        result <= TIE;
                        led_tie <= 1;
                        led_win <= 0;
                        led_lose <= 0;
                    end else if (
                        (p_choice == ROCK && c_choice == SCISSORS) ||
                        (p_choice == PAPER && c_choice == ROCK) ||
                        (p_choice == SCISSORS && c_choice == PAPER)
                    ) begin
                        result <= WIN;
                        led_tie <= 0;
                        led_win <= 1;
                        led_lose <= 0;
                        buzzer <= 0;
                        if (!game_over && player_score < 5) begin
                            player_score <= player_score + 1;
                            if (player_score + 1 == 5) begin
                                game_over <= 1;
                            end
                        end
                    end else begin
                        result <= LOSE;
                        led_tie <= 0;
                        led_win <= 0;
                        led_lose <= 1;
                        if (!game_over && comp_score < 5) begin
                            comp_score <= comp_score + 1;
                            if (comp_score + 1 == 5) begin
                                game_over <= 1;
                            end
                        end
                    end
                end
            endcase
            
            // Buzzer Control Logic
            if (result == LOSE && buzz_cnt < BZ_DUR) begin
                buzzer <= 1;
            end else begin
                buzzer <= 0;
            end
        end
    end
    
    // Next State Logic
    always @(*) begin
        case (state)
            IDLE: begin
                if (db_r || db_p || db_s)
                    next = PLAYER;
                else
                    next = IDLE;
            end
            PLAYER: begin
                next = COMPUTER;
            end
            COMPUTER: begin
                next = RESULT;
            end
            RESULT: begin
                if (~(db_r || db_p || db_s))
                    next = IDLE;
                else
                    next = RESULT;
            end
            default: next = IDLE;
        endcase
    end
endmodule