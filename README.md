# Lex: A Minimalist Lexer Library in Zig

Lex is a helper library for writing simple lexers, primarily for config files.

## Example

```zig
const std = @import("std");
const token = @import("token.zig");

const TokenType = enum {
    colon,
    quoted_string,
    eof,
    newline,
};

const tokens = [_]token.TokenKind(TokenType){
    token.token_kind_single(TokenType, .colon, ':', false),
    token.token_kind_single_ignore(TokenType, ' '),
    token.token_kind_single(TokenType, .newline, '\n', true),
    .{
        .token_type = .quoted_string,
        .token_matcher = struct {
            fn matcher(lexer: *token.Lexer(TokenType)) bool {
                if (lexer.source.ptr[lexer.current_pos] == '\"') {
                    // consume matching token
                    _ = lexer.advance();
                    return true;
                }
                return false;
            }
        }.matcher,
        .token_handler = struct {
            fn handler(lexer: *token.Lexer(TokenType)) void {
                while (lexer.peek() != null and lexer.peek().? != '\n' and lexer.peek().? != '"') {
                    _ = lexer.advance();
                }
                if (lexer.at_end() or lexer.peek() == '\n') {
                    @panic("Unterminated string");
                }
                // The closing "
                _ = lexer.advance();
                // Trim the surrounding quotes
                const value = lexer.source.ptr[lexer.start_pos + 1 .. lexer.current_pos - 1];
                lexer.add_token(.quoted_string, value);
            }
        }.handler,
    },
};

fn eof_handler(_: *token.Lexer(TokenType)) void {}

fn eof_matcher(_: *token.Lexer(TokenType)) bool {
    return false;
}

const eof_token = token.TokenKind(TokenType){
    .token_type = .eof,
    .token_handler = eof_handler,
    .token_matcher = eof_matcher,
};

pub fn main() !void {
    std.debug.print("All your {s} are belong to us.\n", .{"codebase"});

    var gpa = std.heap.DebugAllocator(.{}).init;

    const allocator = std.heap.page_allocator;
    const source_file = try std.fs.openDirAbsolute("/", .{});

    const content = try source_file.readFileAlloc(allocator, "/home/luke/Development/pax/repo/g/glibc/.MANIFEST", std.math.maxInt(u64));

    var lexer = token.Lexer(TokenType).init(gpa.allocator(), content);
    lexer.scan_source(&tokens, .eof);
    std.debug.print("{any}", .{lexer.token_list.items});
}
```
