import pandas as pd
import matplotlib.pyplot as plt

def main():
    df = pd.read_csv("averages2.csv", sep=",") #Argument-stringiksi täytetään SQL-kyselyillä saadun taulun tiedostosijainti
    ax = plt.gca()
    df.plot(kind="line", x="age", y="avg_lvl_active",ax=ax)
    df.plot(kind="line", color="red", x="age", y="avg_lvl_all", ax=ax)
    df.plot(kind="scatter", x="age", y="avg_lvl_active",ax=ax) #Nämä kaksi riviä voi jättää pois jos kuva meinaa näyttää epäsiistiltä
    df.plot(kind="scatter", color="red", x="age", y="avg_lvl_all", ax=ax)
    plt.ylabel("average level")
    plt.savefig('chart.png')


if __name__ == "__main__":
    main()
    