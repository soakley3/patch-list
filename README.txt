# patch-list

[2024:]

https://issues.redhat.com/browse/RHEL-71349
https://lists.crash-utility.osci.io/archives/list/devel@lists.crash-utility.osci.io/thread/FTHEOSEHYHGBW2LU4RFTS7UBKD2ICMWS/

Change check_stack_overflow() to check if the thread_info's cpu
member is smaller than possible existing CPUs, rather than the
kernel table's cpu number (kt->cpus). The kernel table's cpu number
is changed on some architectures to reflect the highest numbered
online cpu + 1. This can cause a false positive in
check_stack_overflow() if the cpu member of a parked task's
thread_info structure, assigned to an offlined cpu, is larger than
the kt->cpus but lower than the number of existing logical cpus.
An example of this is RHEL 7 on s390x or RHEL 8 on ppc64le when
the highest numbered CPU is offlined.

Signed-off-by: Lucas Oakley <soakley(a)redhat.com&gt;
---
 task.c | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)

diff --git a/task.c b/task.c
index 33de7da..93dab0e 100644
--- a/task.c
+++ b/task.c
@@ -11253,12 +11253,12 @@ check_stack_overflow(void)
 				cpu = 0;
 				break;
 			}
-			if (cpu >= kt->cpus) {
+			if (cpu >= get_cpus_present()) {
 				if (!overflow)
 					print_task_header(fp, tc, 0);
 				fprintf(fp, 
 				    "  possible stack overflow: thread_info.cpu: %d >= %d\n",
-					cpu, kt->cpus);
+					cpu, get_cpus_present());
 				overflow++; total++;
 			}
 		}
-- 
2.47.1






[2025:]

https://issues.redhat.com/browse/RHEL-72726
https://lists.crash-utility.osci.io/archives/list/devel@lists.crash-utility.osci.io/thread/HCLRRZAS6WJALKWLCJ4H25NGQ7TJHGJ4/

This simplication fixes the total CPU count being reported
incorrectly in ppc64le and s390x systems when some number of
CPUs have been offlined, as the kt->cpus value is adjusted.
This adds the word "OFFLINE" to the 'sys' output for s390x
and ppc64le, like exists for x86_64 and aarch64 when examining
systems with offlined CPUs.

Without patch:

  KERNEL: /debug/4.18.0-477.10.1.el8_8.s390x/vmlinux
DUMPFILE: /proc/kcore
    CPUS: 1

With patch:

  KERNEL: /debug/4.18.0-477.10.1.el8_8.s390x/vmlinux
DUMPFILE: /proc/kcore
    CPUS: 2 [OFFLINE: 1]

Signed-off-by: Lucas Oakley <soakley(a)redhat.com&gt;
---
 kernel.c | 16 +++++++---------
 1 file changed, 7 insertions(+), 9 deletions(-)

diff --git a/kernel.c b/kernel.c
index 8c2e0ca..3e190f1 100644
--- a/kernel.c
+++ b/kernel.c
@@ -5816,15 +5816,13 @@ display_sys_stats(void)
 				pc->kvmdump_mapfile);
 	}
 	
-	if (machine_type("PPC64"))
-		fprintf(fp, "        CPUS: %d\n", get_cpus_to_display());
-	else {
-		fprintf(fp, "        CPUS: %d", kt->cpus);
-		if (kt->cpus - get_cpus_to_display())
-			fprintf(fp, " [OFFLINE: %d]", 
-				kt->cpus - get_cpus_to_display());
-		fprintf(fp, "\n");
-	}
+        int number_cpus_to_display = get_cpus_to_display();
+        int number_cpus_present = get_cpus_present();
+        fprintf(fp, "        CPUS: %d", number_cpus_present);
+        if (number_cpus_present != number_cpus_to_display)
+                fprintf(fp, " [OFFLINE: %d]",
+                    number_cpus_present - number_cpus_to_display);
+        fprintf(fp, "\n");
 
 	if (ACTIVE())
 		get_xtime(&kt->date);
-- 
2.47.1

