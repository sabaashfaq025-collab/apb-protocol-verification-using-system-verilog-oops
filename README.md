# apb-protocol-verification-using-system-verilog-oops
This APB testbench uses OOP classes for transaction, driver, monitor, and scoreboard. Transactions are transferred between components using mailboxes, while the driver and monitor access the APB interface. The scoreboard compares expected and actual transactions and reports PASS or FAIL.
// Code your design here
// design.sv
//============================================================
// APB INTERFACE
//============================================================

interface apb_if(input logic clk);

    logic        psel;
    logic        penable;
    logic        pwrite;
    logic [31:0] paddr;
    logic [31:0] pwdata;
    logic [31:0] prdata;
    logic        pready;

endinterface


//============================================================
// APB TRANSACTION
//============================================================

class apb_transaction;

    rand bit [31:0] address;
    rand bit [31:0] data;
    rand bit        write;

    function new(
        bit [31:0] address,
        bit [31:0] data,
        bit        write
    );

        this.address = address;
        this.data    = data;
        this.write   = write;

    endfunction


    function void display();

        $display("--------------------------------");
        $display("APB TRANSACTION");
        $display("Address = %h", address);
        $display("Data    = %h", data);
        $display("Write   = %b", write);
        $display("--------------------------------");

    endfunction

endclass


//============================================================
// APB DRIVER
//============================================================

class apb_driver;

    virtual apb_if vif;

    mailbox #(apb_transaction) gen2drv;

    function new(
        virtual apb_if vif,
        mailbox #(apb_transaction) gen2drv
    );

        this.vif     = vif;
        this.gen2drv = gen2drv;

    endfunction


    task run();

        apb_transaction tr;

        forever begin

            gen2drv.get(tr);

            $display("");
            $display("[%0t] DRIVER: Transaction received",
                     $time);

            tr.display();

            // IDLE
            vif.psel    <= 1'b0;
            vif.penable <= 1'b0;
            vif.pwrite  <= 1'b0;

            // SETUP
            @(posedge vif.clk);

            vif.psel    <= 1'b1;
            vif.penable <= 1'b0;
            vif.pwrite  <= tr.write;
            vif.paddr   <= tr.address;
            vif.pwdata  <= tr.data;

            $display("[%0t] DRIVER: SETUP",
                     $time);

            // ACCESS
            @(posedge vif.clk);

            vif.penable <= 1'b1;

            $display("[%0t] DRIVER: ACCESS",
                     $time);

            // Wait for DUT
            wait(vif.pready == 1'b1);

            $display("[%0t] DRIVER: PREADY received",
                     $time);

            // IDLE
            @(posedge vif.clk);

            vif.psel    <= 1'b0;
            vif.penable <= 1'b0;
            vif.pwrite  <= 1'b0;

            $display("[%0t] DRIVER: Transaction complete",
                     $time);

        end

    endtask

endclass


//============================================================
// APB MONITOR
//============================================================

class apb_monitor;

    virtual apb_if vif;

    mailbox #(apb_transaction) mon2scb;

    function new(
        virtual apb_if vif,
        mailbox #(apb_transaction) mon2scb
    );

        this.vif     = vif;
        this.mon2scb = mon2scb;

    endfunction


    task run();

        apb_transaction tr;

        forever begin

            @(posedge vif.clk);

            // Detect SETUP
            if ((vif.psel == 1'b1) &&
                (vif.penable == 1'b0)) begin

                tr = new(
                    vif.paddr,
                    vif.pwdata,
                    vif.pwrite
                );

                $display("[%0t] MONITOR: Transaction detected",
                         $time);

                // Wait for ACCESS
                @(posedge vif.clk);

                wait(vif.pready == 1'b1);

                mon2scb.put(tr);

                $display("[%0t] MONITOR: Sent to scoreboard",
                         $time);

            end

        end

    endtask

endclass


//============================================================
// APB SCOREBOARD
//============================================================

class apb_scoreboard;

    mailbox #(apb_transaction) mon2scb;

    bit [31:0] expected_address;
    bit [31:0] expected_data;
    bit        expected_write;

    function new(
        mailbox #(apb_transaction) mon2scb
    );

        this.mon2scb = mon2scb;

    endfunction


    task run();

        apb_transaction tr;

        forever begin

            mon2scb.get(tr);

            $display("[%0t] SCOREBOARD: Transaction received",
                     $time);

            if ((tr.address == expected_address) &&
                (tr.data    == expected_data) &&
                (tr.write   == expected_write)) begin

                $display("******** SCOREBOARD PASS ********");

            end
            else begin

                $display("******** SCOREBOARD FAIL ********");

                $display("Expected Address = %h",
                         expected_address);

                $display("Actual Address   = %h",
                         tr.address);

                $display("Expected Data    = %h",
                         expected_data);

                $display("Actual Data      = %h",
                         tr.data);

                $display("Expected Write   = %b",
                         expected_write);

                $display("Actual Write     = %b",
                         tr.write);

            end

        end

    endtask

endclass


//============================================================
// APB DUT
//============================================================

module apb_slave(apb_if bus);

    always @(posedge bus.clk) begin

        if (bus.psel && bus.penable) begin

            bus.pready <= 1'b1;

            if (bus.pwrite) begin

                $display("[%0t] DUT: WRITE",
                         $time);

                $display("DUT Address = %h",
                         bus.paddr);

                $display("DUT Data = %h",
                         bus.pwdata);

            end

        end
        else begin

            bus.pready <= 1'b0;

        end

    end

endmodule
// Code your testbench here
// or browse Examples
// test bench .tb
//============================================================
// TOP TESTBENCH
//============================================================

module tb_top;

    logic clk;

    // Clock
    initial begin

        clk = 1'b0;

        forever #5 clk = ~clk;

    end


    // APB interface
    apb_if bus(clk);


    // DUT
    apb_slave dut(bus);


    // Mailboxes
    mailbox #(apb_transaction) gen2drv;
    mailbox #(apb_transaction) mon2scb;


    // Components
    apb_driver     driver;
    apb_monitor    monitor;
    apb_scoreboard scoreboard;


    // Transaction
    apb_transaction tr;


    initial begin

        // Create mailboxes
        gen2drv = new();
        mon2scb = new();


        // Create components
        driver = new(
            bus,
            gen2drv
        );

        monitor = new(
            bus,
            mon2scb
        );

        scoreboard = new(
            mon2scb
        );


        // Start components
        fork

            driver.run();

            monitor.run();

            scoreboard.run();

        join_none


        //====================================================
        // TRANSACTION 1
        //====================================================

        #20;

        scoreboard.expected_address = 32'hA000_0001;
        scoreboard.expected_data    = 32'h1111_1111;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0001,
            32'h1111_1111,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 2
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_0002;
        scoreboard.expected_data    = 32'h2222_2222;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0002,
            32'h2222_2222,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 3
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_0003;
        scoreboard.expected_data    = 32'h3333_3333;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0003,
            32'h3333_3333,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 4
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_0004;
        scoreboard.expected_data    = 32'h4444_4444;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0004,
            32'h4444_4444,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 5
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_0005;
        scoreboard.expected_data    = 32'h5555_5555;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0005,
            32'h5555_5555,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 6
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_0006;
        scoreboard.expected_data    = 32'h6666_6666;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0006,
            32'h6666_6666,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 7
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_0007;
        scoreboard.expected_data    = 32'h7777_7777;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0007,
            32'h7777_7777,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 8
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_0008;
        scoreboard.expected_data    = 32'h8888_8888;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0008,
            32'h8888_8888,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 9
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_0009;
        scoreboard.expected_data    = 32'h9999_9999;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_0009,
            32'h9999_9999,
            1'b1
        );

        gen2drv.put(tr);


        //====================================================
        // TRANSACTION 10
        //====================================================

        #30;

        scoreboard.expected_address = 32'hA000_000A;
        scoreboard.expected_data    = 32'hAAAA_AAAA;
        scoreboard.expected_write   = 1'b1;

        tr = new(
            32'hA000_000A,
            32'hAAAA_AAAA,
            1'b1
        );

        gen2drv.put(tr);


        // Wait for all transactions
        #150;

        $finish;

    end


    //========================================================
    // WAVEFORM
    //========================================================

    initial begin

        $dumpfile("apb_layered.vcd");

        $dumpvars(0, tb_top);

    end

endmodule
