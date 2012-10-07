# mod_fastcgi 2.4.6 chroot patch
# ------------------------------

# Copyright 2012 Rafał Frühling <rafamiga at gmail dot com>

# PURPOSE
# -------

# This patch is for those experiencing problems with mod_fastcgi and chroot. 
# It was written to provide a solution for a weird Apache 2.2 problem which
# manifests itself in "No input file specified" message returned from a
# request for a PHP script located in subdirectory.  My setup was PHP-FPM
# (PHP FastCGI Process Manager) over a socket configured using
# FastCGIExternalServer.

# Weird, ain't it?!

# The configuration triggering this behaviour goes like this:

# DocumentRoot /home/apps/php-app/
#    ...
# FastCgiExternalServer /home/httpd/html/php -idle-timeout 300 -socket /var/run/php-fpm-php.sock
#    ...
# RewriteRule ^/(.*)\.php$ /home/httpd/html/php/$1.php [L]

# "GET /test.php" works flawlessly, but "GET /test2/test3.php" fails with
# "No input file specified" message.  Note: "/home/apps/php-app" is a path
# outside the chroot (actually it's a symlink) and "/home/httpd/html/php" is
# a patch inside chroot (to which the previous symlink well..  links).  And,
# and that's may be the cause of the problem this patch is for, there's no
# "/home/apps/php-app" path in chroot due to it being a r/o filesystem
# (actually it's an ext3 image shared amongst all the vhosts Apache runs).

# This patch replaces SCRIPT_FILENAME environment variable's value with a
# SCRIPT_URL and a user-configurable (chroot) path prepended.  It is done
# for non-top-directory-index requests (i.e.  SCRIPT_URI != /).  Without
# this patch, Apache sets SCRIPT_FILENAME to a DIRECTORY the file's in,
# which PHP-FPM obviously cannot open/process, hence "No input file
# specified" message.

# I still don't know Why Apache behaves this way, but a brief strace session
# revealed that Apache is a) trying to open the chroot path before passing
# the request, along with broken environment, to PHP-FPM socket and b) it
# runs a sub-request before passing it to the socket and in this sub-request
# the environment variable SCRIPT_FILENAME is already "broken".  To find out
# more you can check my post on highoad-php-en Google group:

# http://groups.google.com/group/highload-php-en/browse_thread/thread/4d7663b843b41d77

# This patch would be unneccessary, if Apache allowed this kind of hack:

# RewriteCond %{REQUEST_URI} ^/(.*)\.php$
# RewriteRule ^ - [E=SCRIPT_FILENAME:/home/chroot/WWW/user/%1]

# But it doesn't.

# Any suggestions on why this happens are welcome! Just e-mail me.

# USAGE
# -----

# FastCgiExternalServer takes one extra parameter "-chroot <path>", and the
# value for SCRIPT_FILENAME variable is set to <path> + SCRIPT_URL while
# constructing an array of environment variables passed over the socket to
# PHP-FPM.  Here's an example:

# FastCgiExternalServer /home/httpd/html/php -chroot /home/httpd/html/php  -idle-timeout 300 -socket /var/run/php-fpm-php.sock

# I guess mod_proxy_fcgi (2.4's or to be found on
# http://mproxyfcgi.sourceforge.net/) fixes the problem, but some admins
# still prefer to run mod_fastcgi with Apache 2.2 and mod_fcgi does not have
# an "external PM" feature.  I'd personally go with Nginx instead, which
# allows administrator to set the proper, chrooted value of SCRIPT_FILENAME
# directly in config but my project, for which this patch was developed for,
# was using Apache 2.2.

# As of revision 3 of this patch, you can set FORCE_SCRIPT_FILENAME
# environment variable to override the behaviour described above and set
# SCRIPT_FILENAME regardless of SCRIPT_URL value. Just use this hack as a
# last Rewrite in your virtual host app config:

# RewriteRule ^(.*)$ /home/httpd/html/php/index.php [L,E=FORCE_SCRIPT_FILENAME:/home/httpd/html/php/index.php$1]

# Note: it won't make mod_fastcgi nor PHP-FPM add PATH_INFO environment
# variable to the request but still, a script can guess it from environment
# and in fact eZ Publish, for which this patch was created, manages to get
# the necessary path info.

# NOTES
# -----

# I guess mod_proxy_fcgi fixes the problem, but some admins still prefer to
# run mod_fastcgi with Apache 2.2.  I'd personally go with Nginx instead,
# which allows administrator to set the proper, chrooted value of
# SCRIPT_FILENAME directly in config.

# The spec file is for RHEL/CentOS 5. This is the one I use to make RPMs.
# Notice: remove -DDEBUG=1 -DFCGI_DEBUG=1 from apxs call before making RPM
# for your production server!

# LICENCE: GPL
# ------------
#
# This program is free software; you can redistribute it and/or modify it
# under the terms of the GNU General Public License as published by the Free
# Software Foundation; either version 2 of the License, or (at your option)
# any later version.
# 
# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY
# or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
# for more details.
#    
# you should have received a copy of the GNU General Public License along
# with this program; if not, write to the Free Software Foundation, Inc., 59
# Temple Place - Suite 330, Boston, MA 02111-1307, USA
# 

# RELEASES
# --------

# 2012-10-07, revision 4
#     - includes patch to use PATH_INFO and PATH_TRANSLATED instead of
#       SCRIPT_URL and SCRIPT_FILENAME variable.
# 2012-01-31, revision 3
#     - allow override with FORCE_SCRIPT_FILENAME
# 2012-01-23, revision 2 (first public release)
#     - initial public release

# Homepage: http://orfika.net/src/mod_fastcgi-chroot-patch
# Git: https://github.com/hippich/mod_fastcgi_chroot_patch
