const std = @import("std");

const Allocator = std.mem.Allocator;

const ExecutionError = error{
    MemoryLimitExceeded,
    TimeLimitExceeded,
    InvalidInstruction,
};

const Instruction = enum {
    Add,
    Sub,
    Mul,
    Div,
    Halt,
};

const TraceEntry = struct {
    step: usize,
    acc: i64,
};

const ExecutionContext = struct {
    allocator: Allocator,
    acc: i64 = 0,
    ip: usize = 0,
    steps: usize = 0,
    max_steps: usize,
    trace: std.ArrayList(TraceEntry),

    pub fn init(allocator: Allocator, max_steps: usize) ExecutionContext {
        return .{
            .allocator = allocator,
            .max_steps = max_steps,
            .trace = std.ArrayList(TraceEntry).init(allocator),
        };
    }

    pub fn record(self: *ExecutionContext) !void {
        try self.trace.append(.{
            .step = self.steps,
            .acc = self.acc,
        });
    }

    pub fn deinit(self: *ExecutionContext) void {
        self.trace.deinit();
    }
};

const Program = struct {
    instructions: []Instruction,
    operands: []i64,
};

fn execute(ctx: *ExecutionContext, program: Program) !void {
    while (true) {
        if (ctx.steps >= ctx.max_steps)
            return ExecutionError.TimeLimitExceeded;

        const instr = program.instructions[ctx.ip];
        const operand = program.operands[ctx.ip];

        switch (instr) {
            .Add => ctx.acc += operand,
            .Sub => ctx.acc -= operand,
            .Mul => ctx.acc *= operand,
            .Div => {
                if (operand == 0) return ExecutionError.InvalidInstruction;
                ctx.acc /= operand;
            },
            .Halt => break,
        }

        ctx.steps += 1;
        try ctx.record();
        ctx.ip += 1;
    }
}

fn auditTrace(ctx: *ExecutionContext) void {
    std.debug.print("Execution Trace:\n", .{});
    for (ctx.trace.items) |entry| {
        std.debug.print(
            "Step {d} | Accumulator = {d}\n",
            .{ entry.step, entry.acc },
        );
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const allocator = gpa.allocator();

    var ctx = ExecutionContext.init(allocator, 50);
    defer ctx.deinit();

    const instructions = [_]Instruction{
        .Add,
        .Mul,
        .Sub,
        .Div,
        .Halt,
    };

    const operands = [_]i64{
        10,
        3,
        5,
        5,
        0,
    };

    const program = Program{
        .instructions = instructions[0..],
        .operands = operands[0..],
    };

    try execute(&ctx, program);
    auditTrace(&ctx);

    std.debug.print("Final Accumulator: {d}\n", .{ctx.acc});
}
