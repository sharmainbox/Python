# sftp_zip_files_new
Python Code
import re
import json
import zipfile
import paramiko


def iter_dir_sftp(sftp):
    sub_dirs = []
    files = []
    path = sftp.getcwd()
    path = '/' if path is None else path

    for item in sftp.listdir():
        try:
            sftp.chdir(item)
            sub_dirs.append(item)
            sftp.chdir('..')
        except:
            # Exception occurs when the object traversed is a file not a dir
            if item.endswith('.zip'):
                files.append(item)

    # Iterating for each dir and recursively calling for sub-dir in that dir
    for dirs in sub_dirs:
        try:
            sftp.chdir(dirs)
        except:
            continue

        # Iterating each folder
        iter_dir_sftp(sftp)

        try:
            sftp.chdir('..')
        except:
            continue

    for file in files:
        with sftp.open(file) as zip_file_obj:
            with zipfile.ZipFile(zip_file_obj, 'r') as zip_files:
                file_list_obj = zip_files.namelist()
                no_of_files = 0
                file_list = []
                for ele in file_list_obj:
                    x = zip_files.getinfo(ele)
                    if x.is_dir():
                        continue
                    no_of_files += 1

                    # Reading file data
                    file_obj = zip_files.open(ele)  # Opening file in read binary (rb) format
                    pattern = "Keyword To be Searched"
                    data = file_obj.read(20_097_152)  # Reading first 20 MB data, we can use seek method if pattern
                                                     # present in bottom of file to reduce reading all the file data
                    if re.search(pattern, data.decode('utf-8')):
                        file_list.append({
                            'file_name': x.filename,
                            'file_size': x.file_size,
                            'file_compression_size': x.compress_size
                        })
                print(f"SFTP Path :: {path!r}, Zip File Name :: {file!r}, No of compressed files :: {no_of_files}")
                print(f"{json.dumps(file_list, indent=2)}")
                print('=' * 15)


if __name__ == '__main__':
    # Creating SFTP Connection
    hostname = 'host'
    port = 22
    username = 'user'
    password = 'Password'
    sftp_path = '/path/'

    ssh_client = paramiko.SSHClient()
    ssh_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    ssh_client.connect(hostname=hostname, username=username, password=password, port=port)

    sftp_client = ssh_client.open_sftp()
    print("SFTP Connected")

    sftp_client.chdir(sftp_path)
    iter_dir_sftp(sftp_client)


