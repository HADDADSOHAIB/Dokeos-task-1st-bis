# Project Repository:
https://github.com/HADDADSOHAIB/study-mentors-api

## Introduction:
Ce projet et le backend d'une plateforme qui fait les reservations des sessions entre les etudiants et les formateur. Le front est sur le repository [suivant](https://github.com/HADDADSOHAIB/study-mentors).

Dans les paragraphes suivant, je vais analyser quelque parti du code et en même je propose des amélioration vu que j'ai fait ce code ça fait long et durant cette période j'ai appris beaucoup des chose.

## Authentification:
Pour gerer l'authentification, j'ai utilisé le standard de Json Web Token (JWT), voila la fonction qui fait le sign up dans le controller ([pour plus de details](https://github.com/HADDADSOHAIB/study-mentors-api/tree/master/app/controllers/api/v1)):

```
def create
  account_type = params[:account_type]
  if account_type == 'Student'
    @user = Student.new(user_params)
  elsif account_type == 'Teacher'
    teacher_params = user_params
    teacher_params[:schedule] = {}
    teacher_params[:session_type] = 'online'
    @user = Teacher.new(teacher_params)
  end

  if @user.save
    payload = { user_id: @user.id, account_type: account_type }
    session = JWTSessions::Session.new(payload: payload, refresh_by_access_allowed: true)
    tokens = session.login
    response.set_cookie(
      JWTSessions.access_cookie,
      value: tokens[:access],
      httponly: true,
      secure: Rails.env.production?
    )

    render json: {
      csrf: tokens[:csrf],
      access: tokens[:access],
      current_user: @user,
      categories: []
    }, status: :created
  else
    render json: @user.errors, status: :unprocessable_entity
  end
end
```

Durant la creation d'une compte, on recoit du front le type du compte qu'on veut creer (etudiant ou formateur), puis on creer le token d'authentification et l'envoye dans la reponse.

--> ameliorations: 

```
  account_type = params[:account_type]
  if account_type == 'Student'
    @user = Student.new(user_params)
  elsif account_type == 'Teacher'
    teacher_params = user_params
    teacher_params[:schedule] = {}
    teacher_params[:session_type] = 'online'
    @user = Teacher.new(teacher_params)
  end
```
Pour améliorer cet partie, je propose de faire le meta programming:
```
 @user = account_type.safe_constantize&.new(eval("#{account_type.downcase}_params"))
```
L'avantage de cet appraoch c'est que on evite beacoup des if/else, surtout si on ajoute d'autre type de compte, on a passe de 9 lignes de code à 1 ligne.

## Reservations:

Pour faire des reservations, on tape sur le controller [suivant](https://github.com/HADDADSOHAIB/study-mentors-api/blob/master/app/controllers/api/v1/bookings_controller.rb):
```
def create
  teacher = Teacher.find(create_params[:teacher_id])
  student = Student.find(create_params[:student_id])
  category = Category.find(create_params[:category_id])

  @booking = Booking.new(
    teacher: teacher,
    student: student,
    category: category,
    session_type: create_params[:type],
    date: create_params[:from],
    from: create_params[:from],
    to: create_params[:to]
  )

  if @booking.save
    render json: { booking: @booking }, status: :created
  else
    render json: @booking.errors, status: :unprocessable_entity
  end
end
```
Cet fonction permet de creer des reservations.

--> ameliorations:
```
teacher = Teacher.find(create_params[:teacher_id])
student = Student.find(create_params[:student_id])
category = Category.find(create_params[:category_id])
```
Il y un grand probleme de securité sur cet partie, on ne vérifie pas si le teacher_id et student_id est bien l'id des personne connecte, alors on peut passer des réservations pour d'autre personne, alors ce qu'on doit faire:
1- ajouter le notion de qui est fait la resérvation.
2- comparer l'idee de cette personne avec la id dans le payload.

Dans le [controller globale](https://github.com/HADDADSOHAIB/study-mentors-api/blob/master/app/controllers/application_controller.rb), on a ça:
```
 def current_user
    @current_user ||= if payload['account_type'] == 'Teacher'
                        Teacher.find(payload['user_id'])
                      else
                        Student.find(payload['user_id'])
                      end
  end
```
Alors on peut vérifier qu'est l'utilisateur connecté on utilison @current_user. Ce code aussi peut être améliorer, en effet, nous avons une mauvaise gestion des exceptions, cette fonction:
```
Teacher.find(payload['user_id'])
```
lance une exception s'il ne trouve pas de record dans la base de donnés, la chose qui peuvent cassé la serveur, pour corriger ça, on peut travailler avec:
```
Teacher.find_by(id: payload['user_id'])
```
qui fait la même chose, mais envoyer la valeur nil, s'il le record n'existe pas.