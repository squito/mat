MAT
=======

What is it?
------------

MAT is a tool to *programmatically* work with java heap dumps.  While there are many tools for analyzing
heap dumps, most of them do not have an API to pull out and work with the objects in the heap direclty.
MAT lets you do that.  See the extended example below.

Brief History
--------------

This code was originally pulled directly from https://bitbucket.org/vshor/mat (sorry I converted to git from
mercurial and didn't keep the history because, well, I'm lazy).  It was actually discovered from 
https://github.com/square/haha, but that only works for Android heap dumps.  `vshor/mat` was in turn
taken from the [Eclipse Memory Analyzer Tool](https://eclipse.org/mat) (MAT), but that is unusable outside
of Eclipse.


Example
-------------

My original use case was when I wanted to work with the objects in a heap dump.  Though I could browse the objects
in YourKit, I couldn't work with them programmatically.  In particular, the heap dump I was working with had lots
of 80 MB buffers, and I had no idea what was in them.  A coworker suggested I look for ascii strings in the buffers
to see if that helped shed light on their contents.  I could browse bits of the buffer graphically in YourKit --
but I couldn't do a comprehensive search over 80 MB.

But using MAT, I could pull the byte arrays out of the heap dump, and then work with them directly.  Here's an example
in scala:

```scala
  import java.io.File
  import scala.collection.JavaConverters._

  import org.eclipse.mat.util.IProgressListener
  import org.eclipse.mat.parser.internal.SnapshotFactory
  import org.eclipse.mat.util.IProgressListener.Severity
  import org.eclipse.mat.snapshot.model.IPrimitiveArray

  val factory = new SnapshotFactory()
  val path = "..."

  val listener = new IProgressListener {
    override def sendUserMessage(severity: Severity, s: String, throwable: Throwable): Unit = {
      println(s"$severity: $s $throwable")
    }

    override def isCanceled: Boolean = false

    var workedTotal = 0

    override def done(): Unit = {
      println("done")
    }

    override def worked(i: Int): Unit = {
      workedTotal += i
      println(s"worked $i (total = $workedTotal)")
    }

    override def setCanceled(b: Boolean): Unit = {}

    override def subTask(s: String): Unit = {
      println(s"subtask $s")
    }

    override def beginTask(s: String, i: Int): Unit = {
      println(s"Beginning ${s} with $i units")
    }
  }

  val start = System.currentTimeMillis()
  val snapshot = factory.openSnapshot(new File(path), new java.util.HashMap(), listener)
  val end = System.currentTimeMillis()
  println(s"heap loaded in ${(end - start) / 1000}s")


  val clsName = new Array[Byte](0).getClass().getCanonicalName()
  val icls = snapshot.getClassesByName(clsName, false).asScala.head
  val byteArrayIds = icls.getObjectIds
  val bigByteArrayId = byteArrayIds.maxBy{id => snapshot.getHeapSize(id)}

  val bigByteArray = snapshot.getObject(bigByteArrayId).asInstanceOf[IPrimitiveArray]
  val arr = bigByteArray.getValueArray.asInstanceOf[Array[Byte]]


  /**
    * find runs of bytes that might be ascii characters and print them out to see
    * if they might be meaningful strings
    */
  def findStrings(bytes: Array[Byte], minLength: Int = 8): Unit = {
    var idx = 0
    var inPrintable = false
    var printableBegin = -1
    while (idx < bytes.length) {
      val printable = (bytes(idx) >= 32 && bytes(idx) < 127)
      if (printable && !inPrintable) {
        inPrintable = true
        printableBegin = idx
      } else if (!printable && inPrintable) {
        if (idx - printableBegin >= minLength) {
          val str = new String(bytes.slice(printableBegin, idx).map{_.toChar})
          println(s"""printable from ${printableBegin} to ${idx}: "$str"""")
        }
        inPrintable = false
      }
      idx += 1
    }
  }
  
  findStrings(arr)
```

