pipeline{
    agent any
    stages{
        stage("This is the first stage"){
            steps{
                sh """
                        echo "Welcome here"
                   """
            }
        }
    }
    stages{
        stage("This is the build stage"){
            steps{
                sh """
                        echo "welcome to the production. Added Jenkins"
                        ssh -o StrictHostKeyChecking=no -T -i /var/lib/jenkins/tester2.pem ubuntu@ec2-54-160-154-253.compute-1.amazonaws.com

                        sudo apt update -y

                        cd /var/www

                        sudo rm -rf html

                        git clone https://github.com/monyslim/pix-mix.git html
                   
                   
                   
                   
                   
                   """
            }
        }
    }

}