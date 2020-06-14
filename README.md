Testbed to play with maven relocation experiments for GWT. The basic goal is
 * move GWT to a new groupId so that non-googlers can perform releases
 * with minimal release overlaps so that googler involvement during the overlap 
 * perform releases separately, org.gwtproject first, then com.google.gwt when 
 time permits consists of just shipping poms with no artifacts after the "real" 
 release takes place.
 * downstream libraries and applications can migrate to org.gwtproject when ready,
 and that path should be as straightforward as possible
 
The current approach here then is to test a single overlap release, `org.gwtproject:2.10.0`
and `com.google.gwt:2.10.0`, while also testing with an "old" release, `com.google.gwt:2.10.0`.
The test setup creates these three different releases, with a BOM, and gwt servlet/user/dev 
jars. Each artifact contains a single class unique to that artifactId, so that it will collide
with other versions, if our wiring doesn't correctly remove conflicts.

Several sample libraries exist too, using one (or more?) of those releases in their 
dependencies. Then, applications exist to mix and match gwt versions from their direct and 
transitive dependencies. Each application's pom will validate that there are no collisions on
its classpath.

---


Simple project which is problematic at this time:
 * Depends on `o.g:gwt-user:2.10`
 * Depends on some library which depends on `c.g.g:gwt-user:2.9` (predates the relocation)
 
Such a project will fail, since it seems to depend on both `c.g.g:gwt-user:2.9` and
`o.g:gwt-user:2.10`, and no dependency management asks it to update c.g.g to a relocated
version, so they will conflict.

The naive way to fix this would have been for the project to have a BOM to begin with, but
this still apparently fails:
 * Includes the `o.g:gwt:2.10` bom
 * Depends on `o.g:gwt-user`, managed to `2.10` via the bom
 * Depends on some library which depends on `c.g.g:gwt-user:2.9`.
 
This still seems to fail, since c.g.g is still left at 2.9, so we still don't get the relocation.

Alternative fix which is on its face correct, but not constructive:
 * Includes the `c.g.g:gwt:2.10` bom
 * Depends on `c.g.g:gwt-user`, managed to `2.10` via the bom
 * Depends on some library which depends on `c.g.g:gwt-user:2.9`.
 
This requires that Google continue to release c.g.g BOMs, so isn't an option long term, but
perhaps a year of this could be allowed. It also requires extra complexity to use the ecoystem
for each project, and after Google stops publishing relocated versions or updated BOMs, requires
another phase of updating user applications.

It might be possible to mandate that both the `c.g.g:gwt:2.10` bom _and_ `o.g:gwt:2.future` bom 
be added to a project to enable it to depend on existing stable releases. I have not tested this, 
but even if it does work, at this point it might be better to insist on an explicit `<exclusion>`
on each dependency which might predate this update.

Another approach could be to have c.g.g BOM reference the o.g BOM directly - this might reduce us
to one GWT BOM in an application, but almost certainly requires that it be the c.g.g BOM.
Probably worth testing with an additional o.g release in this project?

---

Another problematic case: 
 * Depends on old gwt (in this case, c.g.g 2.9)
 * Depends on some library which happens to use the new groupId

Recognizing that the GWT project doesn't itself support this use case (that's why we have backward
compatibility _forever_ on stuff that should be long dead like legacy dev mode), there are still
cases where an old version sometimes fixes a problem - bugs happen even after testing. In this case,
imagine that the library is perfectly backward compatible, and only uses new gwt to stay current.
Previously it was safe/suggested to use a bom, but now it appears that an exclusion is the only answer,
or forcing an update to a newer gwt.