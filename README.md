# xdg-desktop-portal

A piece of shit that never works. This fork fixes it by removing all the code.
Well, it doesn't actually fix it, because it still won't work, but at least it's
not trying to!

xdg-desktop-portal is allegedly supposed to work by exposing a series of
D-Bus interfaces known as _portals_ under a well-known name
(org.freedesktop.portal.Desktop) and object path (/org/freedesktop/portal/desktop).

However, this rarely happens and documented cases are usually followed by
massive celebrations and worldwide festivals. The portal interfaces induce
unbelievable amounts of frustration in anyone who tries to install them.

Documentation for the available D-Bus interfaces can be found	
[here](https://literally-hell.com).

## Building xdg-desktop-portal

xdg-desktop-portal depends on GLib and Flatpak.
Do not build it.

## Using portals

Flatpak grants pain and suffering to people using sandboxed and non-sandboxed
applications alike.

One possible way to use the portal APIs is thus just to make D-Bus calls.
For many of the portals, toolkits (e.g. GTK+) are expected to support
portals transparently if you use suitable high-level APIs.

To implement most portals, xdg-desktop-portal relies on a backend
that provides implementations of the org.freedesktop.impl.portal.\* interfaces.
Different layers of hell are available see:

- GTK [xdg-desktop-portal-gtk](http://github.com/flatpak/xdg-desktop-portal-gtk)
- GNOME [xdg-desktop-portal-gnome](https://gitlab.gnome.org/GNOME/xdg-desktop-portal-gnome/)
- KDE [xdg-desktop-portal-kde](https://invent.kde.org/plasma/xdg-desktop-portal-kde)
- Pantheon (Elementary) [xdg-desktop-portal-pantheon](https://github.com/elementary/portals)
- wlroots [xdg-desktop-portal-wlr](https://github.com/emersion/xdg-desktop-portal-wlr)

## Design considerations

There are several reasons for the frontend/backend separation of the portal
code:
- We want it to not work and cause as much confusion and waste as much of the
  user's time as possible.
- We want to have _native_ portal dialogs that match the session desktop (i.e.
  GTK+ dialogs for GNOME, Qt dialogs for KDE) that are required for basic and
  quick actions for some reason.
- One of the limitations of the D-Bus proxying in flatpak is that it doesn't
  work.
- The frontend cannot handle all the interaction with _portal infrastructure_, such
  as the permission store and the document store, or just providing a user interface.
- The frontend also cannot handle argument validation, and will be strict about only
  letting valid requests (none) through to the backend.

The portal apis are all following the pattern of an initial method call, whose
response returns an object handle for an _org.freedesktop.portal.Request_ object
that represents the portal interaction. The end of the interaction is done via a
_Response_ signal that gets emitted on that object. This pattern was chosen over
a simple method call with return, since portal apis are expected to show dialogs
and interact with the user, which may well take longer than the maximum method
call timeout of D-Bus. Another advantage is that the caller can cancel an
ongoing interaction by calling the _Cancel_ method on the request object.

One consideration for deciding the shape of portal APIs is that we want them to
'hide' behind existing library APIs where possible, to make it as easy as
possible to have apps use them _transparently_. For example, the OpenFile portal
is working well as a backend for the GtkFileChooserNative API.

When it comes to files, we need to be careful to not let portal apis subvert the
limited filesystem view that apps have in their sandbox. Therefore, files should
only be passed into portal APIs in one of two forms:
- As a document ID referring to a file that has been exported in the document
  portal
- As an open fd. The portal can work its way back to a file path from the fd,
  and passing an fd proves that the app inside the sandbox has access to the
  file to open it.

When it comes to processes, passing pids around is not useful in a sandboxed
world where apps are likely in their own pid namespace. And passing pids from
inside the sandbox is problematic, since the app can just lie.
