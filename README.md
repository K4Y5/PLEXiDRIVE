# PLEXiDRIVE
Scripts to facilitate the use of Google Drive as storage for Plex media

## Purpose
The purpose of this project is to use Google Drive as a means of storage for Plex.  This project specifically targets using Google Drive unlimited accounts.  Traditionally, using a Drive account with Plex runs into issues with exceeding Google's API call quota. This occurs during Plex scans of large media collections.  To combat this, this project automates the uploading of media to a Drive account and automatically scans the individual directories where new media was placed. This means that only a small subset of the media library will be scanned as opposed to scanning the entire collection (requires automatic Plex scans to be switched off). The scripts also has the ability to upload media to multiple Google accounts for redundancy in a RAID 1-like manner. This can be useful if the Drive accounts have the potential to be banned or revoked (i.e. purchased on eBay, etc.).

## Disclaimer
These scripts are use at your own risk, meaning I am not responsible for any issues or faults that may arise. I have tested these scripts on my own systems and verfied their functionality; however, due diligence is required by the end user. I am in no way affiliated with Google, Plex Inc., or rclone. I am not responsible if a ban is place on the user's Drive account due to abuse or excessive API calls.

## Dependencies
1. [rclone mount](https://rclone.org/commands/rclone_mount/) 
2. [gdrive CLI client](https://github.com/prasmussen/gdrive#downloads)
3. [Plex Media Server](https://support.plex.tv/hc/en-us/articles/200288586-Installation)

## Installation

1. Clone Git repo in home directory
	```bash
	> ~$ git clone https://github.com/masonr/PLEXiDRIVE
	```

2. Edit permissions to allow *plex* user full access
	```bash
	> ~$ sudo chmod -R 777 PLEXiDRIVE
	```

3. Install [rclone](https://rclone.org/install/) and [configure](https://rclone.org/drive/) each Google Drive account

4. Download [gdrive CLI client](https://github.com/prasmussen/gdrive#downloads)

5. Move both rclone and gdrive binaries into a directory found in the PATH environment variable
	```bash
	> ~$ sudo mv gdrive /usr/local/bin/
	> ~$ sudo mv rclone /usr/local/bin/
	> ~$ sudo chown root:root /usr/local/bin/gdrive /usr/local/bin/rclone
	> ~$ sudo chmod 755 /usr/local/bin/gdrive /usr/local/bin/rclone
	```

6. Mount Google Drive(s) using [rclone mount](https://rclone.org/commands/rclone_mount/) with options
	```bash
	> ~$ sudo mkdir /mnt/gdrive-main
	> ~$ sudo rclone mount --allow-non-empty --allow-other gdrive-main:/ /mnt/gdrive-main &
	```
	*Edit path as needed and use rclone remote names configured in Step 3*

7. Link Google Drive account(s) to the gdrive CLI client
	```bash
	> ~$ gdrive --config ~/.gdrive-main about
	```

8. Create media folders within Google Drive accounts and copy folder ID to config file
	* Log in to the Google Drive account
	* Create a new folder with a unique name (i.e. TV Shows, if no other folder is named TV Shows)
	* Enter the new folder
	* Copy the ID found in the URL after "/folders/" (i.e. 0B1uT-U02upTWNUFhRkVSUnBjMU0)
	* Place a new entry into the *gdrive-directory* file
		* For a TV Show root folder, use the form: name1:TV_ROOT::0B1uT-U02upTWNUFhRkVSUnBjMU0
		* For a Movie root folder, use the form: name1:MOVIE_ROOT::0B1uT-U02upTWNUFhRkVSUnBjMU0
		* Where "name1" corresponds to the names given in the *drive_names* parameter
	* Replicate procedure above for each Drive account

9. Determine the Plex media section numbers for the Movies and TV Show libraries
	* Libraries must first be set up on the Plex server (map the Movies library to the rclone mounted path; same for TV Shows)
	```bash
	> ~/PLEXiDRIVE$ sudo su -c 'export LD_LIBRARY_PATH=/usr/lib/plexmediaserver; /usr/lib/plexmediaserver/Plex\ Media\ Scanner --list' plex
		1: Movies
		2: TV Shows
  	```
  	*See command and example output above*
  	* Copy the corresponding library section numbers to the *plexidrive.conf* (plex_movies_section_num & plex_tvshow_section_num)

## Important Notes
* TV Shows must be organized of the form: "(root)/Show Name/Season Number/files"
* The script will not delete empty TV Show folders after successful uploading
* Movies can be placed in individual folders or in the local Movies root directory
* In order to avoid a ban on the Google Drive account with large Plex libraries, the automatic media scans within Plex server settings must be switched off
* It's very important to use the exact notation as described for the *gdrive-directory* entries and the config file parameters or the scripts may not work at all
* The plex-scan script must be run as root user (*sudo ./plex-scan.sh*) as the script must have the effective user as *plex*
* This script uses exponential backoff in cases of the "Rate Limit Exceeded" error

## Usage

### Uploading media
Simply run the script below after configuring the Plex server and setting up the *plexidrive.conf* file
```bash
> ~/PLEXiDRIVE$ ./plexidrive.sh
```

### Scanning Plex library for new files
```bash
> ~/PLEXiDRIVE$ sudo su -c './plex-scan.sh' plex
```

### Cron jobs
In order to automate the uploading of media and Plex scans, cron jobs can be used. Add a cron job to the root crontab for the Plex scan, and to the local user's account for the media uploads.

## Configuration (plexidrive.conf)

### GDrive Settings
* num_of_gdrives: the number of Google Drive accounts to upload media files to
* drive_names: the name(s) of the Google Drive accounts
* gdrive_config_paths: the config path(s) where the gdrive CLI client tokens are stored
* gdrive_mount_paths: where the rclone mount path(s) exist

### Options
* delete_after_upload: denotes if the local media files should be deleted after successful upload
* file_types: the file types to scan for when detecting files to upload

### Plex Library Directories
* plex_tvshow_path: the path of the rclone mounted drive and folder where TV Shows will be found
* plex_movies_path: the path of the rclone mounted drive and folder where Movies will be found
* plex_movies_section_num: the library section number corresponding to Movies, found in installation step 9
* plex_tvshow_section_num: the library section number corresponding to TV Shows, found in installation step 9

### Local Media Directories
* local_tvshow_path: the path where local TV Show media will be found
* local_movies_path: the path where local Movie media will be found

### Enable/Disable Componenets
* enable_show_uploads: enable or disable uploading of TV media
* enable_movie_uploads: enable or disable uploading of Movie media

### **Example Config w/ One Google Drive**
```bash
## GDrive Settings ##
num_of_gdrives=1
drive_names=('gdrive-main')
gdrive_config_paths=('/home/masonr/.gdrive-main')
gdrive_mount_paths=('/mnt/main')

## Options ##
delete_after_upload=true # true/false
file_types="mkv|avi|mp4|m4v|mpg|wmv|flv"

## Plex Library Directories ##
plex_tvshow_path="/mnt/main/TV Shows" # no ending /
plex_movies_path="/mnt/main/Movies" # no ending /
plex_movies_section_num=1
plex_tvshow_section_num=2

## Local Media Directories ##
local_tvshow_path="/home/masonr/tv-shows/" # end with /
local_movies_path="/home/masonr/movies/" # end with /

## Enable/Disable Components ##
enable_show_uploads=true # true/false
enable_movie_uploads=true # true/false
```

### **Example Config w/ Two Google Drives**
```bash
## GDrive Settings ##
num_of_gdrives=2
drive_names=('gdrive-main' 'gdrive-backup')
gdrive_config_paths=('/home/masonr/.gdrive-main' '/home/masonr/.gdrive-backup')
gdrive_mount_paths=('/mnt/main' '/mnt/backup')

## Options ##
delete_after_upload=true # true/false
file_types="mkv|avi|mp4|m4v|mpg|wmv|flv"

## Plex Library Directories ##
plex_tvshow_path="/mnt/main/TV Shows" # no ending /
plex_movies_path="/mnt/backup/Movies" # no ending /
plex_movies_section_num=1
plex_tvshow_section_num=2

## Local Media Directories ##
local_tvshow_path="/home/masonr/tv-shows/" # end with /
local_movies_path="/home/masonr/movies/" # end with /

## Enable/Disable Components ##
enable_show_uploads=true # true/false
enable_movie_uploads=true # true/false
```
