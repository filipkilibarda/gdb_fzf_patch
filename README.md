# gdbfzf
Patch to integrate Fzf fuzzy finder with GDB history search

```diff
diff --git a/readline/readline/emacs_keymap.c b/readline/readline/emacs_keymap.c
index b5e53f4..0e3fc9b 100644
--- a/readline/readline/emacs_keymap.c
+++ b/readline/readline/emacs_keymap.c
@@ -50,7 +50,7 @@ KEYMAP_ENTRY_ARRAY emacs_standard_keymap = {
   { ISFUNC, (rl_command_func_t *)0x0 },		/* Control-o */
   { ISFUNC, rl_get_previous_history },		/* Control-p */
   { ISFUNC, rl_quoted_insert },			/* Control-q */
-  { ISFUNC, rl_reverse_search_history },	/* Control-r */
+  { ISFUNC, rl_fzf_search_history },	/* Control-r */
   { ISFUNC, rl_forward_search_history },	/* Control-s */
   { ISFUNC, rl_transpose_chars },		/* Control-t */
   { ISFUNC, rl_unix_line_discard },		/* Control-u */
diff --git a/readline/readline/funmap.c b/readline/readline/funmap.c
index aaf144d..b7187c6 100644
--- a/readline/readline/funmap.c
+++ b/readline/readline/funmap.c
@@ -127,7 +127,7 @@ static const FUNMAP default_funmap[] = {
   { "quoted-insert", rl_quoted_insert },
   { "re-read-init-file", rl_re_read_init_file },
   { "redraw-current-line", rl_refresh_line},
-  { "reverse-search-history", rl_reverse_search_history },
+  { "reverse-search-history", rl_fzf_search_history },
   { "revert-line", rl_revert_line },
   { "self-insert", rl_insert },
   { "set-mark", rl_set_mark },
diff --git a/readline/readline/isearch.c b/readline/readline/isearch.c
index d6c5904..aa8baf3 100644
--- a/readline/readline/isearch.c
+++ b/readline/readline/isearch.c
@@ -136,6 +136,147 @@ rl_reverse_search_history (int sign, int key)
   return (rl_search_history (-sign, key));
 }
 
+/* If NEW_LINE differs from what is in the readline line buffer, add an
+   undo record to get from the readline line buffer contents to the new
+   line and make NEW_LINE the current readline line. */
+static void
+maybe_make_readline_line (char *new_line)
+{
+  if (new_line && strcmp (new_line, rl_line_buffer) != 0)
+    {
+      rl_point = rl_end;
+
+      rl_add_undo (UNDO_BEGIN, 0, 0, 0);
+      rl_delete_text (0, rl_point);
+      rl_point = rl_end = rl_mark = 0;
+      rl_insert_text (new_line);
+      rl_add_undo (UNDO_END, 0, 0, 0);
+    }
+}
+
+char *
+escape_quotes(char *s)
+{
+  char c;
+  size_t len_s  = strlen(s);
+  size_t len_ss = len_s*4 + 1;		// Large enough that no resizing necessary
+  char *ss = (char *) calloc(len_ss, 1);
+
+  if (ss == NULL) {
+    printf("Failed to alloc string");
+    exit(1);
+  }
+
+  size_t i = 0;
+  size_t j = 0;
+  while (1) {
+    if (i == len_s) {
+      break;
+    }
+    c = s[i];
+    if (c == '\'') {
+      strcpy(&ss[j], "'\\''");
+      j += 4;
+    } else {
+      ss[j] = c;
+      j += 1;
+    }
+    i += 1;
+  }
+
+  return ss;
+}
+
+int
+rl_fzf_search_history (int sign, int key)
+{
+  int outbound[2];
+  int inbound[2];
+
+  if (pipe(outbound) || pipe(inbound)) {
+    return -1;
+  }
+
+  int parent_read  = inbound[0];
+  int parent_write = outbound[1];
+  int child_read   = outbound[0];
+  int child_write  = inbound[1];
+
+  pid_t child = fork();
+  if (child == 0) {
+    // In child
+    dup2(child_read, 0); 	// setup stdin
+    dup2(child_write, 1); 	// setup stdout
+
+    close(parent_read);
+    close(parent_write);
+
+    char *shellcmd;
+    char *escaped_line = escape_quotes(rl_line_buffer);
+
+    // TODO read0 and print0 options should be hardcoded, not in the shell command
+    if (asprintf(&shellcmd, "fzf $GDB_FZF_OPTS --query='%s'", escaped_line) == -1) {
+      puts("Failed to allocate string");
+      exit(-1);
+    }
+
+    free(escaped_line);
+
+    char *argv[] = { "/bin/sh", "-c", shellcmd, NULL };
+    execve(argv[0], argv, environ);
+    exit(-1);			// if execve returns it failed
+  }
+
+  close(child_read);
+  close(child_write);
+
+  rl_crlf();
+
+  HIST_ENTRY **hlist = history_list();
+
+  FILE *in  = fdopen(parent_read, "r");
+  FILE *out = fdopen(parent_write, "w");
+
+  setvbuf(in,  NULL, _IONBF, 0);
+  setvbuf(out, NULL, _IOFBF, 4096);
+
+  HIST_ENTRY *hentry;
+  for (int i = 0; hlist && (hentry = hlist[i]); i++) {
+    fputs(hentry->line, out);
+    fputc(0, out);
+  }
+
+  fclose(out);
+
+  #define MAX_CMD_LEN 256
+  char gdb_cmd[MAX_CMD_LEN];
+  gdb_cmd[MAX_CMD_LEN-1] = '\0';
+
+  int c;
+  for (int i = 0; i < MAX_CMD_LEN-1; i++) {
+    c = fgetc(in);
+    if (c == EOF) {  		// Failed to read until a null byte so abort with an empty command
+      gdb_cmd[0] = '\0';
+      break;
+    }
+    gdb_cmd[i] = (char) c;
+    if (c == '\0') break;
+  }
+  
+  // Wait for the child to terminate. If it doesn't terminate, the program will just hang and we'll
+  // know there's an issue with the child not exiting.
+  waitpid(child, NULL, 0);
+
+  // Don't bother checking for a success return code b/c sometimes we kill it w/ C-c
+
+  maybe_make_readline_line(gdb_cmd);
+
+  // THIS IS PROBABLY WHAT i WANT B/C NO CLEAR LINE TERMCAP
+  rl_forced_update_display();
+
+  return 0; // XXX Guessing ret val
+}
+
 /* Search forwards through the history looking for a string which is typed
    interactively.  Start with the current line. */
 int
diff --git a/readline/readline/readline.h b/readline/readline/readline.h
index da78271..3406e93 100644
--- a/readline/readline/readline.h
+++ b/readline/readline/readline.h
@@ -183,6 +183,7 @@ extern int rl_paste_from_clipboard PARAMS((int, int));
 
 /* Bindable commands for incremental searching. */
 extern int rl_reverse_search_history PARAMS((int, int));
+extern int rl_fzf_search_history PARAMS((int, int));
 extern int rl_forward_search_history PARAMS((int, int));
 
 /* Bindable keyboard macro commands. */
```


